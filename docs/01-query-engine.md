# 对话引擎（Query Engine）

## 概述

对话引擎是 Claude Code Harness 的核心循环，负责将用户输入、工具结果、Memory 组装成 API 请求，驱动与 LLM 的多轮对话，并处理工具调用、上下文压缩、Token 预算等所有运行时逻辑。它是 Harness 与 LLM 之间的粘合层，位于 UI 层（React/Ink）和 API 接缝（`query.ts`）之间。

---

## 核心职责

- 管理单次会话的完整生命周期：从用户输入到最终结果返回
- 驱动「流式 API 调用 → 工具执行 → 再次调用」的多轮循环
- 在每轮迭代前执行上下文预处理：snip、microcompact、autocompact、context collapse
- 管理 Token Budget，在输出接近上限时自动触发 continuation 或提前终止
- 每轮结束后执行 Stop Hooks（含 TeammateIdle、TaskCompleted）
- 维护跨轮的会话状态：消息数组、文件缓存、用量统计、权限拒绝记录

---

## 架构概览

```
submitMessage(prompt)              ← QueryEngine 入口（SDK/headless 路径）
  │
  ├─ fetchSystemPromptParts()      ← 系统提示词组装（context.ts + queryContext.ts）
  ├─ processUserInput()            ← 斜杠命令处理、附件注入
  ├─ recordTranscript()            ← 持久化用户消息（crash-safe）
  │
  └─ query(params)                 ← 进入 query.ts 主循环
       │
       └─ queryLoop()  [while(true)]
            │
            ├─ applyToolResultBudget()     ← 工具结果大小限制
            ├─ snipCompactIfNeeded()       ← 历史 snip（HISTORY_SNIP feature）
            ├─ microcompact()              ← 微压缩
            ├─ contextCollapse()           ← 上下文折叠（CONTEXT_COLLAPSE feature）
            ├─ autocompact()              ← 自动全量压缩
            │
            ├─ callModel()  [streaming]   ← 实际 API 请求（claude.ts）
            │    └─ for await message     ← 流式消费 assistant/tool_use 块
            │
            ├─ runTools() / StreamingToolExecutor   ← 工具并发执行
            │
            ├─ handleStopHooks()          ← Stop / TeammateIdle / TaskCompleted hooks
            │
            ├─ checkTokenBudget()         ← Token Budget 决策
            │
            └─ continue / return          ← 决定下一轮迭代还是终止
```

数据流方向：
`用户输入` → `系统提示词 + 历史消息` → `API 请求` → `流式响应` → `工具执行` → `工具结果追加` → `下一轮 API 请求`

---

## 核心实现

### QueryEngine — 主循环入口

**文件**：`src/QueryEngine.ts`（1295 行）

`QueryEngine` 是一个有状态的类，**每个会话对应一个实例**，跨多个 `submitMessage()` 调用保持状态。

```typescript
export class QueryEngine {
  private mutableMessages: Message[]        // 跨轮持久消息数组
  private abortController: AbortController  // 中止信号
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录（SDK 上报用）
  private totalUsage: NonNullableUsage       // 累计 token 用量
  private discoveredSkillNames: Set<string>  // 当前轮次发现的 skill（轮次级别清零）
  private loadedNestedMemoryPaths: Set<string>  // 已加载的嵌套 Memory 路径
}
```

`submitMessage(prompt)` 的关键步骤：

1. **系统提示词组装**：调用 `fetchSystemPromptParts()`，合并 `defaultSystemPrompt`、`customSystemPrompt`、`appendSystemPrompt`、Memory mechanics prompt（当 `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 被设置时）。
2. **用户输入处理**：`processUserInput()` 处理斜杠命令，返回 `shouldQuery`（是否需要 API 调用）。若 `shouldQuery=false`（纯本地命令），直接返回结果，不进入 `query()`。
3. **转录持久化**：在进入 API 循环前 `recordTranscript()`，确保进程崩溃后仍可 `--resume`。
4. **进入 query 循环**：`for await (const message of query({ messages, systemPrompt, ... }))` 消费所有事件，同步持久化 assistant/user/compact_boundary 消息。
5. **结果汇总**：循环结束后 yield `SDKResult`，包含 `total_cost_usd`、`usage`、`permission_denials`、`stop_reason` 等。

权限拒绝追踪通过包装 `canUseTool`：每次拒绝都 push 到 `this.permissionDenials`，最终随 result 上报给 SDK 调用方。

---

### query.ts — LLM API 接缝与主循环

**文件**：`src/query.ts`（1729 行）

`query()` 是对外导出的生成器函数，内部委托给 `queryLoop()`。

```typescript
export async function* query(params: QueryParams): AsyncGenerator<..., Terminal>
```

`queryLoop()` 是真正的 `while(true)` 循环，每次迭代代表一轮「API 调用 + 工具执行」。

**循环内每次迭代的执行顺序：**

1. **上下文预处理**（按顺序，均可能裁减 `messagesForQuery`）：
   - `applyToolResultBudget()`：对超大工具结果做内容替换
   - `snipCompactIfNeeded()`：历史 snip（feature gate: `HISTORY_SNIP`）
   - `microcompact()`：微压缩（cached microcompact 延迟到 API 响应后生效）
   - `contextCollapse.applyCollapsesIfNeeded()`：上下文折叠（feature gate: `CONTEXT_COLLAPSE`）
   - `autocompact()`：自动全量压缩，触发后 `messagesForQuery` 替换为压缩后摘要

2. **阻断检查**：若 token 数超过硬性上限且 autocompact 未启用，yield 错误并返回 `blocking_limit`。

3. **API 调用**：`deps.callModel()` 流式消费 `AssistantMessage`，收集 `toolUseBlocks`。
   - Fallback 模型：若主模型失败触发 `FallbackTriggeredError`，对已流出的 orphaned messages yield tombstone，然后用 fallback 模型重试。
   - `max_output_tokens` 恢复：错误被暂扣，先尝试 escalate 到 64k token，再最多重试 `MAX_OUTPUT_TOKENS_RECOVERY_LIMIT`（3）次，注入 meta 消息 "Output token limit hit. Resume directly…"。
   - `prompt_too_long`（413）恢复：先尝试 context collapse drain，再尝试 reactive compact，均失败则 surface 错误。

4. **工具执行**：`runTools()` 或 `StreamingToolExecutor`（feature gate: `streamingToolExecution`）并发执行所有 `toolUseBlocks`，结果追加到 `toolResults`。

5. **Stop Hooks**：`handleStopHooks()` — 见下节。

6. **Token Budget 检查**：`checkTokenBudget()` — 见下节。

7. **循环决策**：若有 `toolUseBlocks` 且未被阻止，继续下一轮；否则返回 `Terminal`（`reason: 'completed'` 等）。

**State 对象**（跨迭代可变）：

```typescript
type State = {
  messages: Message[]
  toolUseContext: ToolUseContext
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number
  hasAttemptedReactiveCompact: boolean
  maxOutputTokensOverride: number | undefined
  pendingToolUseSummary: Promise<ToolUseSummaryMessage | null> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined   // 上一次 continue 的原因，供测试断言
}
```

每个 continue 站点写 `state = { ... }` 全量替换，而不是 9 个单独赋值，保持不变量清晰。

---

### context.ts — 系统提示词生成

**文件**：`src/context.ts`（190 行）

提供两个 `memoize` 包裹的异步函数，**整个会话只计算一次**：

- **`getSystemContext()`**：返回 `{ gitStatus?, cacheBreaker? }`
  - `gitStatus`：并发执行 `git branch/status/log/config user.name`，截断超过 2000 字符的 status，格式化为 system-reminder 块注入给模型
  - `cacheBreaker`：`BREAK_CACHE_COMMAND` feature 启用时，注入 `[CACHE_BREAKER: ...]` 破坏 prompt cache（调试用）
  - 在 CCR（`CLAUDE_CODE_REMOTE`）或禁用 git instructions 时跳过 git status

- **`getUserContext()`**：返回 `{ claudeMd?, currentDate }`
  - `claudeMd`：遍历目录树读取所有 `CLAUDE.md` 文件，过滤注入的 memory 文件，合并内容
  - `currentDate`：`Today's date is YYYY-MM-DD.`
  - `--bare` 模式且无 `--add-dir` 时跳过 CLAUDE.md 自动发现

这两个 context 在 `query.ts` 的 `appendSystemContext()` 调用中拼接到 `systemPrompt` 末尾，并通过 `prependUserContext()` 注入到消息数组头部（便于 prompt cache 分层）。

---

### Token Budget 管理

**文件**：`src/query/tokenBudget.ts`

用于 `+500k` 超长输出场景的自动续写控制（feature gate: `TOKEN_BUDGET`，与 `task_budget` API 参数不同）。

```typescript
const COMPLETION_THRESHOLD = 0.9   // 达到预算 90% 才停止
const DIMINISHING_THRESHOLD = 500  // 连续两轮增量 < 500 token 视为边际收益递减
```

`BudgetTracker` 跟踪：`continuationCount`、`lastDeltaTokens`、`lastGlobalTurnTokens`、`startedAt`。

`checkTokenBudget()` 决策逻辑：
- 若 `agentId` 存在（子 agent）或 `budget === null`，直接返回 `stop`
- 若用量 < 90% 且未触发边际递减：返回 `continue`，附带 nudge 消息（由 `getBudgetContinuationMessage()` 生成），注入为 meta user message 驱动下一轮
- 若触发边际递减（连续 3 轮且每轮增量 < 500）或用量 ≥ 90%：返回 `stop`，记录 `completionEvent` 供 analytics 上报

---

### Stop Hooks

**文件**：`src/query/stopHooks.ts`

`handleStopHooks()` 是一个异步生成器，在每轮「模型完成且无工具调用」后触发，返回 `{ blockingErrors, preventContinuation }`。

**执行顺序：**

1. **后台任务**（非阻塞，fire-and-forget）：
   - `executePromptSuggestion()`：下一条提示建议
   - `executeExtractMemories()`：Memory 提取（feature gate: `EXTRACT_MEMORIES`）
   - `executeAutoDream()`：Auto Dream（background fork）
   - `cleanupComputerUseAfterTurn()`：Computer Use 清理（feature gate: `CHICAGO_MCP`）
   - 以上在 `--bare` 模式下全部跳过

2. **Job 分类**（阻塞，60s 超时）：`TEMPLATES` feature 启用且在 job 目录中运行时，`classifyAndWriteState()` 更新 `state.json`，确保 `claude list` 不显示陈旧状态。

3. **Stop Hooks 执行**：`executeStopHooks()` 运行用户配置的 stop hook 脚本，收集：
   - `blockingErrors`：阻塞性错误，会作为 user message 注入并触发下一轮（`transition: 'stop_hook_blocking'`）
   - `preventContinuation`：阻止继续，直接返回 `stop_hook_prevented`
   - 非阻塞错误通过 notification 提示用户（`ctrl+o` 展开）

4. **Teammate 专属 Hooks**（仅 `isTeammate()` 时）：
   - `executeTaskCompletedHooks()`：对当前 teammate 所有 `in_progress` 任务逐一触发
   - `executeTeammateIdleHooks()`：触发 idle 通知（让 team lead 知道 teammate 空闲）

---

## 关键设计决策

**1. `while(true)` + State 对象替换，而非递归**

`queryLoop` 用可变 `State` 对象 + `continue` 跳转代替递归，避免深度 agent 场景下的调用栈溢出，且所有 continue 原因都记录在 `state.transition` 供测试断言。

**2. 上下文压缩分层**

snip → microcompact → contextCollapse → autocompact 四层按顺序执行，每层都可以独立生效，越靠后越激进。collapse 优先于 autocompact 的意图是：能用细粒度折叠解决就不做全量摘要。

**3. 错误暂扣（withhold）机制**

`max_output_tokens` 和 `prompt_too_long`（413）错误在流式阶段被暂扣，不立即 yield 给调用方，等待恢复逻辑决定是否重试。若 SDK 调用方提前收到 error 消息，会终止会话导致恢复逻辑无法运行。

**4. Tombstone 消息**

fallback 模型切换时，已流出的 orphaned assistant messages 通过 yield `{ type: 'tombstone', message }` 通知 UI 和 transcript 删除，避免带有无效 thinking block 签名的消息残留在上下文中。

**5. task_budget vs TOKEN_BUDGET**

两套预算机制并存：`task_budget`（API 层参数，控制整个 agentic turn 的输出 token 总量）和 `TOKEN_BUDGET`（feature gate，控制超长输出时的自动续写行为），逻辑完全独立。

**6. Transcript 写入时机**

用户消息在进入 `query()` 循环**之前**持久化，而非等到 API 响应。这样即使进程在等待 API 响应时被杀死，`--resume` 也能从用户消息处恢复，而不是丢失整个 turn。

---

## 与其他系统的交互

- **工具系统**：`query.ts` 通过 `runTools()` / `StreamingToolExecutor` 执行工具，工具结果作为 `user` 消息追加到 `messagesForQuery`，触发下一轮 API 调用。`canUseTool` 是注入的权限检查函数，`QueryEngine` 在其外层包装了拒绝追踪。

- **状态管理（AppState）**：`QueryEngine` 持有 `getAppState` / `setAppState` 回调，工具执行后可更新 `toolPermissionContext`（如 `alwaysAllowRules`）、`fileHistory`、`attribution` 等字段。`AppState` 是 React 状态，变更会触发 UI 重渲染。

- **Memory 系统**：`context.ts` 的 `getUserContext()` 读取 `CLAUDE.md` 文件链（`getMemoryFiles()` + `getClaudeMds()`）注入系统提示词；`stopHooks.ts` 在每轮结束后 fire-and-forget 调用 `executeExtractMemories()` 提取新记忆写回 Memory 文件；`loadMemoryPrompt()` 在 SDK 自定义提示词 + Memory path override 时注入 Memory 操作指令。
