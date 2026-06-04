# 状态管理（State Management）

## 概述

Claude Code 使用自研的极简 Store 模式管理全局状态，以 `AppState` 为唯一数据源，驱动 React/Ink UI 重渲染。Store 的设计绕开了 Redux 等重型方案，直接用 `useSyncExternalStore` 订阅切片，配合 `onChangeAppState` 响应式监听器处理副作用（CCR 同步、auth 缓存清理等）。

## 核心职责

- 以 `AppState`（单一扁平对象）存储所有运行时状态：消息列表、任务、权限、设置、MCP 连接、speculation 等
- `createStore` 提供 `getState`/`setState`/`subscribe` 三件套，`Object.is` 检测变更防止无效重渲染
- `onChangeAppState` 响应特定字段变化（`toolPermissionContext.mode` 等）同步到 CCR / SDK 外部状态
- `selectors.ts` 提供派生状态计算（`getViewedTeammateTask`、`getActiveAgentForInput`）
- `useAppState(selector)` → `useSyncExternalStore`，只在 selector 选中值变化时重渲染

## 架构概览

```
AppStateProvider（React 树根部）
  ↓ createStore(initialState, onChangeAppState)
  → Store { getState, setState, subscribe, listeners: Set<Listener> }
  ↓
AppStoreContext（React Context）
  ↓
useAppState(selector) → useSyncExternalStore(subscribe, getSnapshot)
  → 仅当 Object.is(old, new) === false 时重渲染

setState(updater) 流程：
  prev → updater(prev) → next
  Object.is(next, prev) → 无操作（防抖）
  否则：state = next
    → onChangeAppState({ newState, oldState }) [副作用：CCR/SDK 同步]
    → listeners.forEach(notify)                [触发 Ink 重渲染]
```

## 核心实现

### Store — 极简状态容器

**文件**：`src/state/store.ts`

```ts
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

function createStore<T>(initialState, onChange?) {
  let state = initialState
  const listeners = new Set<Listener>()
  return {
    getState: () => state,
    setState: updater => {
      const next = updater(state)
      if (Object.is(next, state)) return  // 无变化，短路
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },
    subscribe: listener => { listeners.add(listener); return () => listeners.delete(listener) }
  }
}
```

**设计要点**：
- 不可变更新（返回新对象），`Object.is` 浅比较即可检测变更
- `listeners` 用 `Set` 而非数组，`delete` 为 O(1)
- `onChange` 钩子用于副作用，与 listeners 严格分离

### AppState — 核心字段结构

**文件**：`src/state/AppStateStore.ts`

AppState 是一个大而扁的对象，分为两部分：

**`DeepImmutable<{...}>` 部分（递归只读）**：
```ts
type AppState = DeepImmutable<{
  settings: SettingsJson           // ~/.claude/settings.json 内容
  verbose: boolean
  mainLoopModel: ModelSetting      // 当前主循环模型
  mainLoopModelForSession: ModelSetting // session 锁定模型（不受中途切换影响）
  toolPermissionContext: ToolPermissionContext // 权限上下文（mode/rules）
  expandedView: 'none' | 'tasks' | 'teammates' // Ctrl+O 展开视图
  kairosEnabled: boolean           // Assistant 模式开关
  speculation: SpeculationState    // 推测执行状态
  promptSuggestion: { text, promptId, shownAt, acceptedAt, ... }
  notifications: { current, queue }
  elicitation: { queue }           // MCP elicitation 请求队列
  inbox: { messages }              // 文件 mailbox 消息
  sessionHooks: SessionHooksState  // 运行时注册的 function hooks
  thinkingEnabled: boolean | undefined
  promptSuggestionEnabled: boolean
  // ... 60+ 其他字段（bridge 状态、swarm 状态、插件、tmux 面板等）
}>
```

**裸对象部分**（含函数类型，无法 DeepImmutable）：
```ts
& {
  tasks: { [taskId: string]: TaskState }         // 所有后台任务
  agentNameRegistry: Map<string, AgentId>        // Agent 名称→ID 映射
  foregroundedTaskId?: string                    // 前台任务 ID
  viewingAgentTaskId?: string                    // 正在查看的 teammate transcript
  mcp: { clients, tools, commands, resources, pluginReconnectKey }
  plugins: { enabled, disabled, commands, errors, installationStatus, needsRefresh }
  agentDefinitions: AgentDefinitionsResult
  fileHistory: FileHistoryState
  attribution: AttributionState
  todos: { [agentId: string]: TodoList }
  swarmState?: { selfAgentName, isLeader, teammates: {...} }
}
```

**Speculation 状态**（inline 于 AppState）：
```ts
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string
      abort: () => void
      startTime: number
      messagesRef: { current: Message[] }       // 可变 ref，避免每条消息 spread
      writtenPathsRef: { current: Set<string> } // 已写入 overlay 的相对路径
      boundary: CompletionBoundary | null
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean
      pipelinedSuggestion?: { text, promptId, generationRequestId }
    }
```

### AppStateProvider — React 集成层

**文件**：`src/state/AppState.tsx`

`AppStateProvider` 包裹整个 React 树根部：
- 一次性调用 `createStore(initialState, onChangeAppState)`，存入 `useState`（稳定引用）
- `useSettingsChange` 监听 settings 文件变更，通过 `applySettingsChange` 更新 store
- mount 时检查 `isBypassPermissionsModeDisabled()`，自动降级 bypass 权限模式

`useAppState(selector)` 用法：
```ts
// 订阅单个值
const verbose = useAppState(s => s.verbose)
// 订阅子对象（必须是已有引用，不可返回新对象）
const { text, promptId } = useAppState(s => s.promptSuggestion)
```

`useSetAppState()` 返回稳定的 `store.setState` 引用，不订阅任何状态（调用方不重渲染）。

### onChangeAppState — 副作用响应器

**文件**：`src/state/onChangeAppState.ts`

每次 `setState` 后（有实际变更时）同步触发，用于将内部状态变化同步到外部系统：

1. **`toolPermissionContext.mode` 变更** → `notifySessionMetadataChanged`（同步 CCR）+ `notifyPermissionModeChanged`（SDK 状态流）
   - 内部模式（bubble/ungated auto）先 externalize 再比较，避免噪音
   - Ultraplan 模式变化携带 `isUltraplan` 标志

2. **settings 变更** → `updateSettingsForSource`（写回磁盘）

3. **auth 变更** → `clearApiKeyHelperCache`、`clearAwsCredentialsCache` 等缓存清理

**为什么集中在这里**：权限模式有 8+ 个变更路径（Shift+Tab 循环、plan 命令、bridge 远程控制、ExitPlanModePermissionRequest 等），如果各处分散通知，很容易遗漏。`onChangeAppState` 作为唯一副作用出口保证了一致性。

### selectors.ts — 派生状态

**文件**：`src/state/selectors.ts`

纯函数，无副作用，仅计算派生值：

- `getViewedTeammateTask(appState)` → 获取正在查看的 teammate task（返回 `InProcessTeammateTaskState | undefined`）
- `getActiveAgentForInput(appState)` → 决定用户输入路由目标：
  - `{ type: 'leader' }` — 发往主 Agent
  - `{ type: 'viewed', task }` — 发往被查看的 teammate
  - `{ type: 'named_agent', task }` — 发往具名子 Agent

### pendingMessages 内存队列

**文件**：`src/utils/messageQueueManager.ts`

不在 AppState 中，而是独立的内存队列（`pendingMessages`），用于任务完成通知在 LLM 下一轮前插入 `messages[]`：
- `enqueuePendingNotification({ value, mode, priority, agentId })`：入队
- `dequeueNextPendingNotification(agentId)`：出队（按 priority：`'next'` > `'later'`）
- QueryEngine 在发 API 请求前调用出队，将任务完成 XML 插入消息列表

## 关键设计决策

1. **自研极简 Store 而非 Redux/Zustand**：Claude Code 的状态更新路径清晰（绝大多数是 `setAppState(prev => ({ ...prev, key: value }))`），不需要 action/reducer 抽象，`Object.is` 比较足够高效。

2. **`DeepImmutable<T>` 强制不可变性**：核心 AppState 字段用 `DeepImmutable` 包裹，TypeScript 编译期防止意外直接 mutation，确保 Store 的 `Object.is` 检测有效。`tasks` 等含函数类型的字段只能裸对象，是有意为之的例外。

3. **`useSyncExternalStore` 而非 `useContext` + `useState`**：`useSyncExternalStore` 是 React 18 推荐的外部 Store 集成方式，保证并发模式下不出现 tearing（撕裂）。selector 模式让组件只订阅自己用到的片段，避免全量重渲染。

4. **`tasks` 不进 DeepImmutable**：TaskState 包含 `shellCommand`（含 fd）、`AbortController`、函数回调等不可序列化的字段，无法 deep-freeze，单独保留为裸对象区域。

## 与其他系统的交互

- **Ink UI 层**：`AppStateProvider` 包裹 React 树，所有组件通过 `useAppState(selector)` 订阅状态，`setState` 触发重渲染
- **Task 系统**：`LocalShellTask`、`LocalAgentTask` 等通过 `updateTaskState` 写入 `AppState.tasks`
- **Hook 系统**：`useCanUseTool` 读 `AppState.toolPermissionContext`；session hooks 存 `AppState.sessionHooks`
- **Bridge / CCR**：`onChangeAppState` 检测 mode 变化后调 `notifySessionMetadataChanged` 同步到远端
- **Settings**：`useSettingsChange` 监听文件系统变化，`applySettingsChange` 更新 store 中的 `settings` 字段
- **Speculation**：`AppState.speculation` 存储当前推测执行状态，`startSpeculation`/`abortSpeculation` 通过 `setAppState` 驱动
