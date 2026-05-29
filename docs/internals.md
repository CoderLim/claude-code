# Claude Code 内部机制分析

源码来自 2026 年 3 月 npm sourcemap 泄露事件，基于泄露代码的逆向分析。

---

## 整体架构

### CC = Harness + LLM

```
Claude Code = Harness + LLM
```

**LLM**：Claude 模型本身，负责推理和决策（调哪个 tool、生成回复）。

**Harness**：除 LLM 之外的所有运行时基础设施：
- 把用户输入、工具结果、memory 组装成 messages 喂给 LLM
- 执行 LLM 调用的 tool（bash、文件读写、MCP…）
- 管理权限、hooks、tasks、UI 渲染

接缝在 `query.ts`——唯一真正发 API 请求的地方。`QueryEngine.ts` 在外面跑循环，把 harness 和 LLM 粘在一起。

### 三层模型

```
UI 层       → React/Ink（REPL、组件、对话框）
Harness 层  → QueryEngine、tools、hooks、tasks、permissions
LLM 层      → Anthropic API（query.ts）
```

### 启动序列

```
main.tsx
  startMdmRawRead()        // MDM 配置并行读
  startKeychainPrefetch()  // keychain 并行读
  init()                   // 认证、trust dialog
  fetchBootstrapData()     // 远程配置
  initializeGrowthBook()   // feature flags
  launchRepl()             // 渲染 App + REPL
    → QueryEngine 就绪，等待用户输入
```

---

## Task 系统

后台任务类型（`src/tasks/`）：

| Task 类型 | 用途 |
|---|---|
| `LocalShellTask` | Bash 命令后台执行 |
| `LocalAgentTask` | 本地子 Agent 进程 |
| `RemoteAgentTask` | 远程 Agent（ULTRAPLAN / 云端） |
| `InProcessTeammateTask` | 同进程内的 teammate |
| `LocalWorkflowTask` | 工作流编排（`WORKFLOW_SCRIPTS` flag） |
| `MonitorMcpTask` | MCP 监控任务（`MONITOR_TOOL` flag） |
| `DreamTask` | 后台记忆整合（autoDream） |

### LocalWorkflowTask

`WORKFLOW_SCRIPTS` feature flag 保护，实现文件不在泄露代码里。从引用可知：

- task 类型 `'local_workflow'`，短标识 `'w'`
- 状态栏显示 "1 background workflow"
- 有对应的 `WorkflowTool`（Claude 可主动触发）
- 也注册为 slash command（`/workflow-name`，补全时标注 `(workflow)`）
- 子 Agent transcript 归组到 `subagents/workflows/<runId>/`

Workflow 与 Skill 的区别：

| | Skill | Workflow |
|---|---|---|
| 本质 | Markdown 指令，Claude 读后执行 | 脚本文件，harness 直接运行 |
| 执行者 | Claude 推理 | harness 进程 |

---

## Hooks 系统

用户在 `settings.json` 配置的 shell 命令钩子，harness 在生命周期节点自动触发，**不经过 Claude 推理**。

支持事件：

```
SessionStart / SessionEnd
PreToolUse / PostToolUse / PostToolUseFailure
UserPromptSubmit / Stop / StopFailure
SubagentStart / SubagentStop
PreCompact / PostCompact
TaskCreated / TaskCompleted / TeammateIdle
PermissionRequest / PermissionDenied
```

三种执行方式：`execAgentHook`、`execHttpHook`、函数回调。

---

## Agent 间通信

### 通信类型总览

```
同进程 teammate  → AppState pendingMessages（内存）
跨进程 teammate  → 文件 Mailbox（磁盘 + 文件锁）
本地跨工具       → UDS socket
远程 IDE/手机    → bridge session
Terminal pane    → tmux/iTerm2 命令注入
```

### 1. 文件 Mailbox（主要机制）

路径：`~/.claude/teams/{team}/inboxes/{agent_name}.json`

用文件锁（lockfile + retry backoff）保证并发安全。`SendMessageTool` 底层调 `writeToMailbox()`。

### 2. In-process 消息队列

直接写入 `AppState.tasks[id].pendingMessages[]`，不经过文件系统。用于 `InProcessTeammateTask`。

### 3. SendMessageTool 四路路由

| `to` 格式 | 路由 |
|---|---|
| `teammate-name` | 写入文件 mailbox |
| `*` | broadcast 给所有 team 成员 |
| `uds:<socket-path>` | Unix Domain Socket（本机） |
| `bridge:<session-id>` | Remote Control bridge（跨机） |

结构化协议消息（JSON 消息体）：
- `shutdown_request` / `shutdown_response`（approve/reject）
- `plan_approval_response`

### 4. UDS（Unix Domain Socket）

`UDS_INBOX` feature flag 保护。同机器两个独立 Claude Code 进程互发消息。
- 只支持纯文本
- 自动允许（无需用户确认）
- 发送方通过 `ListPeers` 发现 socket 路径

### 5. Bridge（Remote Control）

跨机器，经 Anthropic CCR v2 服务器中转。
- 必须先连 Remote Control（`/remote-control`）
- **永远需要用户确认**（`safetyCheck`，bypassPermissions 也绕不过）
- 只支持纯文本

### ListPeers

`UDS_INBOX` flag 保护。Session 注册表查询工具：

每个 Claude Code 进程启动时在 `~/.claude/sessions/{pid}.json` 写入：
```json
{
  "pid": 12345,
  "sessionId": "sess_01...",
  "cwd": "/path/to/project",
  "messagingSocketPath": "/tmp/cc-socks/12345.sock",
  "bridgeSessionId": "bridge_01..."
}
```

`ListPeers` 扫描目录，验活（`isProcessRunning(pid)`），返回可寻址的 session 列表。同一进程 UDS 和 bridge 都有时只暴露 UDS（本地优先）。

---

## /diff 命令

**交互式 TUI diff 查看器**，用 React/Ink 渲染。

两种数据源（`←/→` 切换）：
1. **Current** — `git diff HEAD`（当前未提交改动）
2. **Turn N** — 当前 session 中第 N 轮 Claude 的文件改动

底层：
- `useDiffData` hook → `git diff HEAD --numstat` + `git diff HEAD` + `git ls-files --others`
- `useTurnDiffs` hook → 遍历 messages 数组提取每轮改动
- 两层视图：list（文件列表）/ detail（单文件 hunks）
- merge/rebase/cherry-pick 状态下不展示（`isInTransientGitState()`）

---

## /btw 命令

**"顺便问一句"**，不打断主对话的旁路提问。

核心机制：
- `immediate: true`：不等主对话停止，键入即触发
- `runForkedAgent()` → 独立 API 调用，并行运行
- **无工具**（`canUseTool` 强制 deny）
- **最多 1 轮**（`maxTurns: 1`）
- `skipCacheWrite: true`：不写 cache entry

**完全隔离，不污染主 agent**：
- 问题和回答从不进入 `context.messages`
- `setAppState` 是 no-op
- `display: 'skip'` → dismiss 时不向主对话插入任何内容
- 结果只写 sidechain transcript（旁路存储，不回注主 session）

**Cache 命中优化**：优先读 `getLastCacheSafeParams()`——由 `handleStopHooks` 在每轮结束后保存的字节完全相同的参数，保证 prompt cache 命中。

---

## Speculation（推测执行）

用户打字时，在后台**预执行**用户可能提交的下一条 prompt，提交时直接采用结果，跳过等待。

> 仅对 Anthropic 内部员工开启：`process.env.USER_TYPE === 'ant'`

### 整体流程

```
用户上一轮结束
  → PromptSuggestion 生成预测文本
  → startSpeculation(suggestionText)
      → runForkedAgent() 后台并行执行
          → Read/Glob/Grep 等只读工具：允许
          → 文件写入：重定向到 overlay 目录
          → Bash 写操作 / 权限不足：停下，记 boundary
用户输入 ≠ 预测 → abortSpeculation()，丢弃
用户输入 = 预测 → handleSpeculationAccept()
  → overlay 文件复制回真实目录
  → speculation messages 注入主 messages[]
  → boundary = complete → 不再发 API 请求
  → boundary ≠ complete → 从 boundary 点继续发一次请求
```

### 文件 Overlay

写操作不落主文件系统，落在 `~/.claude/tmp/speculation/{pid}/{id}/`。

**Copy-on-write**：写之前先把原文件复制到 overlay，再在 overlay 上写。接受时 `copyOverlayToMain()` 批量覆盖；拒绝时 `safeRemoveOverlay()` 删除。

### 工具许可

| 工具 | 处理 |
|---|---|
| Read / Glob / Grep / LSP | 允许 |
| Edit / Write / NotebookEdit | `acceptEdits`/`bypassPermissions` 模式下允许（写 overlay）；否则停，记 `edit` boundary |
| Bash（只读） | 允许 |
| Bash（写/cd） | 停，记 `bash` boundary |
| 其他 | deny，记 `denied_tool` boundary |

### Boundary 类型

```ts
type CompletionBoundary =
  | { type: 'complete'; outputTokens }
  | { type: 'bash'; command }
  | { type: 'edit'; toolName; filePath }
  | { type: 'denied_tool'; toolName; detail }
```

### Pipelining

speculation 完成后立即生成下一条预测并启动下一轮 speculation，形成连续预执行链。

---

## Prompt Cache

Anthropic 服务端把相同前缀的 attention KV 计算结果缓存，下次请求命中相同前缀时直接复用，省去重新推理的时间和成本。

### Attention KV 是什么

Transformer 的每个 token 被投影成 Query/Key/Value 三个向量：

```
Attention(Q, K, V) = softmax(Q·Kᵀ / √d) · V
```

**KV Cache**：逐 token 生成时，已计算过的 K、V 矩阵存起来，下次直接读，不重算。

**Prompt Cache = KV Cache 的跨请求持久化**：把 KV 矩阵序列化存储，下次独立请求如果前缀相同直接加载。前缀任意一个 token 不同，从那位置往后的 KV 全部失效。

### cache_control 标记

```ts
getCacheControl() → { type: 'ephemeral', ttl?: '1h', scope?: 'global' }
```

TTL 两档：
- 默认 5 分钟
- 1h TTL：ant 员工 + 未超限的 claude.ai 订阅用户，由 GrowthBook `tengu_prompt_cache_1h_config` 控制，session 启动时**锁定**（防止 overage 状态变化导致 TTL 翻转炸缓存）

### addCacheBreakpoints

每次请求只放**一个** cache_control 标记：

```ts
// 正常请求：放最后一条消息（写入新缓存前缀）
markerIndex = messages.length - 1

// skipCacheWrite（fire-and-forget fork）：放倒数第二条
// 不让 fork 的尾巴污染服务端 KV cache
markerIndex = messages.length - 2
```

### CacheSafeParams 模式

`/btw`、speculation、promptSuggestion 等 forked agent 全部复用主线程的参数：

```ts
type CacheSafeParams = {
  systemPrompt   // 字节完全相同
  userContext
  systemContext
  toolUseContext // 含 tools 列表、thinkingConfig
  forkContextMessages
}
```

由 `handleStopHooks` 每轮结束后保存，fork 直接复用 → 字节完全相同 = 保证命中缓存。

### skipCacheWrite 语义

fire-and-forget fork（/btw、speculation）设 `skipCacheWrite: true`：只读缓存，不产生新 cache entry，不污染主线程缓存前缀。

### Cache Break 检测（`promptCacheBreakDetection.ts`）

两阶段：
1. **Phase 1（发请求前）**：`recordPromptState()` 对 system prompt、tools、model、betas 等全部哈希，与上轮比对，记录 `pendingChanges`
2. **Phase 2（收响应后）**：`checkResponseForCacheBreak()` 检查 `cache_read_input_tokens` 是否比上轮掉了 5% 以上且超过 2000 tokens

能检测原因包括：system prompt 变化、tool schema 变化、model 切换、fast mode 切换、TTL 过期（5min/1h）、betas 变化、effort 变化等。写 diff 文件到 `~/.claude/tmp/cache-break-xxxx.diff` 供调试。

---

## Monitor 机制

用于监听长时间运行进程的 stdout，每行输出触发一次通知。

与 `run_in_background` 的区别：

| | `run_in_background`（Bash） | Monitor |
|---|---|---|
| 通知时机 | 进程退出时一次 | 每行 stdout 一次 |
| 设计意图 | 等待完成 | 持续流式事件 |

### 两种形态

**`kind: 'monitor'` on `LocalShellTask`**（已有代码）：

| | `kind: 'bash'` | `kind: 'monitor'` |
|---|---|---|
| Stall watchdog | ✅ | ❌（设计为长跑） |
| 结束通知文本 | `"xxx" completed` | `Monitor "xxx" stream ended` |
| 通知合并 | ✅ 多个合并 | ❌ 独立显示 |
| 通知优先级 | `'later'` | `'next'` |

**`MonitorMcpTask`**（`MONITOR_TOOL` flag 保护，实现文件不在泄露代码里）：
- 独立 task 类型 `'monitor_mcp'`，短标识 `'m'`
- Agent 退出时调 `killMonitorMcpTasksForAgent()` 清理

### 输出模式

Claude Code 进程管理有两种输出模式：

| 模式 | 实现 | Node.js 流？ |
|---|---|---|
| Bash 后台命令（文件模式） | stdout 直接写文件 fd，绕过 JS | ❌ 轮询文件尾部（1s 间隔） |
| Hooks（pipe 模式） | `StreamWrapper` + `Readable.on('data')` | ✅ |
| Monitor（推断） | pipe 模式 + 逐行切割 + 逐行通知 | ✅ |

**BashTool sleep 拦截**：`MONITOR_TOOL` 开启后，BashTool 拦截 `sleep N` 命令并报错，强迫 Claude 用正确工具——轮询/日志监听用 Monitor，不要用 sleep loop。
