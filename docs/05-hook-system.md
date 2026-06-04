# Hook 系统（Hook System）

## 概述

Hook 系统是 Claude Code 的两层可扩展机制：一层是 React hooks（前缀 `use`）驱动 Ink UI 的状态管理和副作用；另一层是用户在 `settings.json` 中配置的 shell 命令钩子，在特定生命周期节点由 Harness 自动触发。两者名称相同但完全独立，本文重点覆盖两层的设计和 `useCanUseTool` 权限检查流程。

## 核心职责

- **settings.json shell 钩子**：在 26 个生命周期事件（`PreToolUse`、`PostToolUse`、`SessionStart` 等）触发，执行用户定义的 shell/agent/HTTP 脚本，不经过 Claude 推理
- **`useCanUseTool`**：工具执行前的权限检查 React hook，聚合 deny 规则、auto-approve 规则、分类器（`BASH_CLASSIFIER`）、UI 确认弹窗等多路决策
- **`useGlobalKeybindings`**：注册全局快捷键（Ctrl+T/O/E/C 等）
- **`useIDEIntegration`**：检测并自动连接 IDE 插件（VSCode/JetBrains 等），注入 MCP 配置
- **`useCanUseTool` + Coordinator/Swarm handler**：在多 Agent 场景下路由权限决策

## 架构概览

```
两层 Hook 体系：

[层 1：React hooks（运行时状态/副作用）]
src/hooks/use*.tsx / src/hooks/use*.ts
  - useCanUseTool       权限检查
  - useGlobalKeybindings 快捷键注册
  - useIDEIntegration   IDE 自动连接
  - useCommandQueue     命令队列处理
  - useInboxPoller      文件 mailbox 轮询
  - 60+ 其他 UI/状态 hooks

[层 2：settings.json shell 钩子（生命周期回调）]
~/.claude/settings.json or .claude/settings.json
  hooks:
    PreToolUse:    [{ matcher: 'Bash', hooks: [{ type: 'command', command: '...' }] }]
    PostToolUse:   [...]
    SessionStart:  [...]

执行流程（shell 钩子）：
HookEvent 触发点
  ↓
hooksConfigSnapshot.getHooks(event, matcher)
  ↓ 三种执行方式
  ├── type: 'command'  → execShellHook (子进程 + 5min 超时)
  ├── type: 'agent'    → execAgentHook (LLM multi-turn query)
  └── type: 'http'     → execHttpHook (axios POST + SSRF guard)
  ↓
SyncHookJSONOutput / AsyncHookJSONOutput
  → { continue, decision, updatedInput, additionalContext, ... }
```

## 核心实现

### 26 个支持的 Hook 事件

**文件**：`src/entrypoints/sdk/coreTypes.ts`

```ts
const HOOK_EVENTS = [
  'PreToolUse',        // 工具执行前（可 approve/block/修改 input）
  'PostToolUse',       // 工具执行后（可修改 MCP 工具输出）
  'PostToolUseFailure',// 工具执行失败后
  'Notification',      // 系统通知触发
  'UserPromptSubmit',  // 用户提交 prompt 前（可追加 context）
  'SessionStart',      // session 启动（可设置 watchPaths）
  'SessionEnd',        // session 结束
  'Stop',              // Claude 停止生成
  'StopFailure',       // Claude 停止异常
  'SubagentStart',     // 子 Agent 启动
  'SubagentStop',      // 子 Agent 停止
  'PreCompact',        // compact 开始前
  'PostCompact',       // compact 完成后
  'PermissionRequest', // 权限确认弹窗（可 allow/deny/修改权限）
  'PermissionDenied',  // 权限被拒绝（可重试）
  'Setup',             // 初始化阶段
  'TeammateIdle',      // Teammate 进入 idle
  'TaskCreated',       // Task 创建
  'TaskCompleted',     // Task 完成
  'Elicitation',       // MCP elicitation 请求
  'ElicitationResult', // MCP elicitation 结果
  'ConfigChange',      // 配置变更
  'WorktreeCreate',    // Worktree 创建
  'WorktreeRemove',    // Worktree 删除
  'InstructionsLoaded',// CLAUDE.md 加载完毕
  'CwdChanged',        // 工作目录变更
  'FileChanged',       // 被监听文件变更（需 SessionStart 设置 watchPaths）
]
```

### 三种执行方式

**文件**：`src/utils/hooks/execAgentHook.ts`、`execHttpHook.ts`、`execShellHook.ts`

**1. shell 命令钩子**（`type: 'command'`）：
- 最常见，执行用户定义的 shell 脚本
- 超时默认 10 分钟（`TOOL_HOOK_EXECUTION_TIMEOUT_MS`）
- stdin 接收 JSON 化的 hook 输入（tool name、input、cwd 等）
- stdout 解析为 `SyncHookJSONOutput`

**2. agent 钩子**（`type: 'agent'`，`execAgentHook`）：
- 启动一次独立的 LLM multi-turn query
- 使用 `getSmallFastModel()` 小模型执行
- `hook.prompt` 中 `$ARGUMENTS` 替换为 JSON 化 hook 输入
- 输出 `SyntheticOutputTool` 结构化结果

**3. HTTP 钩子**（`type: 'http'`，`execHttpHook`）：
- axios POST 到用户指定 URL
- SSRF 防护：`ssrfGuardedLookup` 过滤内网 IP
- 沙箱模式下通过代理路由（sandbox network proxy）
- 响应 body 解析为 `SyncHookJSONOutput`

**4. 函数回调钩子**（`type: 'function'`，session 内部）：
- 非用户配置，由代码（如 schema 模式）动态注册
- 存储在 `AppState.sessionStore.hooks` Map（避免 O(N²) 复制）
- 接收 `messages[]`，返回 `boolean`（pass/block）

### Hook 响应结构

```ts
type SyncHookJSONOutput = {
  continue?: boolean        // false → 停止当前操作（附 stopReason 给用户看）
  suppressOutput?: boolean  // true → 隐藏 hook stdout 不显示到 transcript
  stopReason?: string       // 显示给用户的停止原因
  decision?: 'approve' | 'block'  // PreToolUse 专用
  reason?: string           // 决策原因
  systemMessage?: string    // 警告消息（展示给用户）
  hookSpecificOutput?: ...  // 各事件专属字段：
    // PreToolUse: permissionDecision, updatedInput, additionalContext
    // PostToolUse: updatedMCPToolOutput, additionalContext
    // SessionStart: initialUserMessage, watchPaths
    // PermissionRequest: { behavior: 'allow'/'deny', updatedInput, updatedPermissions }
    // PermissionDenied: { retry: true }
    // FileChanged: watchPaths（更新监听路径）
}
```

异步模式：`{ async: true, asyncTimeout?: number }` → hook 后台运行，主流程不等待。

### useCanUseTool — 权限检查流程

**文件**：`src/hooks/useCanUseTool.tsx`

```
工具调用发起
  ↓
hasPermissionsToUseTool(tool, input, ctx) → PermissionDecision
  ↓ 
  allow → 直接通过（规则匹配：alwaysAllowRules / isReadOnly 等）
  deny  → 拒绝（alwaysDenyRules / TRANSCRIPT_CLASSIFIER auto-mode）
  ask   → 需要用户确认
    ↓
    awaitAutomatedChecksBeforeDialog?
      → handleCoordinatorPermission   协调者模式：转发到 worker 队列
      → handleSwarmWorkerPermission   Swarm worker：广播到 leader
    ↓
    BASH_CLASSIFIER 推测性分类器检查（speculative prefetch）
      → 高置信度匹配已知安全规则 → allow（跳过弹窗）
    ↓
    handleInteractivePermission → 弹 PermissionRequest UI 等待用户
      → 用户 allow/deny/always-allow/always-deny
```

`CanUseToolFn` 签名：
```ts
type CanUseToolFn = (
  tool, input, toolUseContext, assistantMessage, toolUseID,
  forceDecision?  // speculation 用来复用已有决策
) => Promise<PermissionDecision<Input>>
```

**TRANSCRIPT_CLASSIFIER（Auto Mode）**：
- `result.decisionReason.type === 'classifier'` 且 `classifier === 'auto-mode'` → auto deny
- 触发 `recordAutoModeDenial`，UI 显示 `"xxx denied by auto mode · /permissions"`

### useGlobalKeybindings — 全局快捷键

**文件**：`src/hooks/useGlobalKeybindings.tsx`

注册以下全局快捷键（底层用 `useKeybinding` hook + keybindings.json 配置）：
- `Ctrl+T`：切换 todo 列表视图
- `Ctrl+O`：切换 transcript 模式
- `Ctrl+E`：显示全部 messages（transcript 模式下）
- `Ctrl+C` / `Escape`：退出 transcript 模式
- 其他 TERMINAL_PANEL 相关快捷键（feature flag 保护）

读 `AppState.expandedView`，通过 `setAppState` 切换 screen 状态。

### useIDEIntegration — IDE 自动连接

**文件**：`src/hooks/useIDEIntegration.tsx`

检测以下触发条件（任一满足则自动连接）：
- `globalConfig.autoConnectIde === true`
- `--auto-connect-ide` CLI flag
- `isSupportedTerminal()`（检测 iTerm2、Terminal.app 等）
- `CLAUDE_CODE_SSE_PORT` 环境变量
- 用户请求安装 IDE 扩展

连接成功后通过 `setDynamicMcpConfig` 注入 MCP 配置，IDE 工具（文件选择、LSP 等）作为 MCP server 注册。

### Session Hook 存储优化

**文件**：`src/utils/hooks/sessionHooks.ts`

`AppState.sessionStore.hooks` 使用 `Map` 而非 `Record`：
- `Map.set()` 为 O(1)，不变更引用 → `Object.is(next, prev)` 为 true → 跳过 store listeners
- 高并发 Swarm 场景下，N 个并行 Agent 同时注册 function hook 的代价从 O(N²)（Record spread）降为 O(N)

## 关键设计决策

1. **三种 hook 执行方式并存**：shell 命令适合轻量检查；agent 适合需要推理的复杂决策；HTTP 适合外部系统集成。统一接口（`SyncHookJSONOutput`）让调用方无感。

2. **`useCanUseTool` 分层决策**：deny rules → allow rules → 自动化检查（classifier/coordinator）→ 用户 UI 确认，每层都有短路路径，避免不必要的弹窗打扰。

3. **`FileChanged` hook + watchPaths 动态注册**：SessionStart 时声明监听路径，文件变更后触发 hook，让用户可以实现"文件改动时自动处理"的工作流，不依赖轮询。

4. **异步 hook 模式**：`{ async: true }` 允许 hook 后台运行，不阻塞 tool 执行，适合日志记录、审计等不需要影响决策的场景。

## 与其他系统的交互

- **QueryEngine**：在 `PreToolUse` / `PostToolUse` 等节点调用 `runHooks(event, ...)`，hook 输出可修改 tool input 或 block 执行
- **工具系统**：`canUseTool`（来自 `useCanUseTool`）注入每个 `tool.call()`，决定是否允许执行
- **AppState / Store**：`useCanUseTool` 通过 `getAppState()` 读 `toolPermissionContext`；session hooks 存 `AppState.sessionStore.hooks`
- **Skills 系统**：`registerSkillHooks` 在 Skill 调用时注册 frontmatter 声明的 hooks（`skill.hooks` 字段）
- **Coordinator/Swarm**：`handleCoordinatorPermission`、`handleSwarmWorkerPermission` 在多 Agent 场景下将权限决策路由到正确的 Agent
