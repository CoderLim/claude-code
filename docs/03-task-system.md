# 任务系统（Task System）

## 概述

任务系统管理 Claude Code 中所有长时间或后台运行的操作，包括 Shell 命令、子 Agent 进程、远程 Agent、同进程 teammate、记忆整合 Agent 和 MCP 监控器。它让 LLM 可以在不阻塞主对话的情况下异步触发外部工作，并在任务完成时通过通知机制将结果回注到对话上下文。

## 核心职责

- 定义 7 种 Task 类型并统一管理其生命周期（pending → running → completed/failed/killed）
- 提供 `TaskStateBase` 基础状态结构和 `generateTaskId` 前缀机制
- 维护 stdout 磁盘缓冲，支持进程输出轮询（LocalShellTask）
- 通过 `enqueuePendingNotification` 将完成事件异步回送给 LLM
- 集成到 AppState，所有状态变更通过 `updateTaskState` 原子更新

## 架构概览

```
BashTool / AgentTool / DreamTask
  ↓ spawnShellTask / registerAsyncAgent / registerDreamTask
Task 注册表（AppState.tasks[]）
  ├── LocalShellTask       stdout → 磁盘文件 → 1s 轮询
  ├── LocalAgentTask       子 Agent 进程（ProgressTracker）
  ├── RemoteAgentTask      远程 Agent（CCR v2 轮询）
  ├── InProcessTeammateTask AsyncLocalStorage 隔离同进程
  ├── DreamTask            记忆整合 forked Agent
  ├── LocalWorkflowTask    工作流脚本（WORKFLOW_SCRIPTS flag）
  └── MonitorMcpTask       MCP 监控器（MONITOR_TOOL flag）
  ↓ 完成/失败
enqueuePendingNotification → pendingMessages → LLM 下一轮
```

任务 ID 前缀：`b`（bash）、`a`（agent）、`r`（remote）、`t`（teammate）、`w`（workflow）、`m`（monitor）、`d`（dream），后跟 8 位随机小写字母数字。

## 核心实现

### Task 基础接口

**文件**：`src/Task.ts`

```ts
type Task = {
  name: string
  type: TaskType
  kill(taskId: string, setAppState: SetAppState): Promise<void>
}

type TaskStateBase = {
  id: string          // 前缀+8位随机，如 b3f7a2c9d1e0...
  type: TaskType
  status: TaskStatus  // 'pending' | 'running' | 'completed' | 'failed' | 'killed'
  description: string
  toolUseId?: string  // 关联的 LLM tool_use block ID
  startTime: number
  endTime?: number
  totalPausedMs?: number
  outputFile: string  // stdout 磁盘文件路径
  outputOffset: number
  notified: boolean   // 防重复通知标志
}
```

7 种 TaskType：`'local_bash'`、`'local_agent'`、`'remote_agent'`、`'in_process_teammate'`、`'local_workflow'`、`'monitor_mcp'`、`'dream'`

终态判断：`isTerminalTaskStatus(status)` → `completed | failed | killed`

### LocalShellTask — Shell 命令后台执行

**文件**：`src/tasks/LocalShellTask/LocalShellTask.tsx`

Bash 命令在前台执行超过 `ASSISTANT_BLOCKING_BUDGET_MS`（15s）后调用 `spawnShellTask` 转为后台任务。

**stdout 写文件 fd（绕过 JS 流）**：
- `shellCommand.background(taskId)` 将 stdout 直接 fd-redirect 到磁盘文件，不经过 Node.js Stream
- 主线程通过 `tailFile(outputPath, ...)` 轮询文件尾部（1s 间隔）读取增量输出

**Stall Watchdog**（非 monitor 类型）：
- 每 5s 检查文件 size 变化
- 若 45s 无增长且最后一行匹配 prompt 特征（`(y/n)`、`Press Enter` 等）→ 发高优先级通知告知 LLM 命令卡在交互式输入
- monitor 类型跳过 watchdog（设计为长跑）

**通知差异**（bash vs monitor）：
- bash：`BACKGROUND_BASH_SUMMARY_PREFIX + "xxx" completed/failed/killed`，优先级 `'later'`
- monitor：`Monitor "xxx" stream ended/failed/stopped`，优先级 `'next'`

### LocalAgentTask — 本地子 Agent

**文件**：`src/tasks/LocalAgentTask/LocalAgentTask.tsx`

通过 `registerAsyncAgent` 注册，`runAgent` 驱动执行。

**进度追踪**（`ProgressTracker`）：
- `toolUseCount`：已调用工具次数
- `latestInputTokens`：最新一次输入 token（API 累计值，取最新）
- `cumulativeOutputTokens`：输出 token 累计（逐轮求和）
- `recentActivities`：最近 5 个工具调用记录（用于 UI 显示"正在做什么"）

支持前台/后台切换：`registerAgentForeground` / `unregisterAgentForeground`；前台 Agent 完成时关联 tool_use 直接返回结果，后台 Agent 通过通知回送。

### RemoteAgentTask — 远程 Agent

**文件**：`src/tasks/RemoteAgentTask/RemoteAgentTask.tsx`

通过 Anthropic CCR v2（Teleport API）在远端运行 Agent，本地轮询事件流（`pollRemoteSessionEvents`）。

支持的远程任务类型（`RemoteTaskType`）：`'remote-agent'`、`'ultraplan'`、`'ultrareview'`、`'autofix-pr'`、`'background-pr'`。

特殊字段：
- `isLongRunning`：不在首次 result 后标记 completed（持续运行型任务）
- `ultraplanPhase`：ULTRAPLAN 特有阶段（`needs_input` / `plan_ready`），显示在 UI 徽章
- `reviewProgress`：ultrareview 场景的 scanner 进度（finding/verifying/synthesizing）

### InProcessTeammateTask — 同进程 Teammate

**文件**：`src/tasks/InProcessTeammateTask/InProcessTeammateTask.tsx`

使用 Node.js `AsyncLocalStorage` 隔离，在同一进程内运行独立 Agent 实例，共享内存但有独立的 context 和 setAppState。

- 通过 `AppState.tasks[id].pendingUserMessages[]` 接收主线程发来的消息（内存队列，不经过文件系统）
- `appendTeammateMessage`：记录 transcript 供 zoomed view 展示
- `injectUserMessageToTeammate`：允许用户向正在运行的 teammate 发消息
- 支持 `shutdownRequested` 标志实现优雅终止（区别于 kill）
- 团队标识：`agentName@teamName`

### DreamTask — 自动记忆整合

**文件**：`src/tasks/DreamTask/DreamTask.ts`

autoDream 触发时（对话轮数/tokens 达到阈值），启动后台 forked Agent 执行记忆整合，DreamTask 使后台 Agent 在 UI footer 可见。

**整合流程（4 阶段 prompt）**：`orient`（定位）→ `gather`（收集相关历史）→ `consolidate`（整合到 MEMORY.md）→ `prune`（清理旧条目）

**状态字段**：
- `phase`：`'starting'` | `'updating'`（首次 Edit/Write tool_use 后翻转）
- `sessionsReviewing`：本次整合覆盖的历史 session 数
- `filesTouched`：观察到被 Edit/Write 修改的文件路径（非完整，仅工具调用可见的部分）
- `turns`：最近 30 条 Dream Agent 的 assistant 回复（工具调用折叠为计数）
- `priorMtime`：kill 时用于回滚 consolidation lock 时间戳

### 通知机制

**文件**：`src/utils/messageQueueManager.ts`

任务完成/失败后调用 `enqueuePendingNotification()`，消息格式为 XML 标签结构：

```xml
<task-notification>
  <task-id>b3f7a2c9</task-id>
  <tool-use-id>toolu_xxx</tool-use-id>
  <output-file>/tmp/cc-output/b3f7a2c9.txt</output-file>
  <status>completed</status>
  <summary>Background command "npm test" completed (exit code 0)</summary>
</task-notification>
```

通知优先级：
- `'next'`：下一轮输入前插入（Monitor 完成、stall 检测触发）
- `'later'`：合并到下次用户输入（普通 bash 完成）

`notified` 标志在 `updateTaskState` 原子设置，防止 kill 和自然完成竞态导致重复通知。

## 关键设计决策

1. **stdout 写磁盘 fd 而非 Node Stream**：绕过 JS 事件循环，避免高速输出时内存积压。轮询读文件尾部的方式虽有 1s 延迟，但内存可控，适合长时命令。

2. **任务状态全量存 AppState**：所有 TaskState 都在 `AppState.tasks` 中，UI 层直接 selector 订阅，任务进度实时反映在 footer 和 Shift+Down 弹窗，无需额外状态同步。

3. **InProcessTeammate 用 AsyncLocalStorage 而非进程隔离**：启动快、通信无序列化开销，适合需要频繁交互的 teammate 场景。代价是共享 heap，teammate 崩溃会影响主进程。

4. **DreamTask 限制 `turns` 为最近 30 条**：记忆整合可能跑很多轮，限制 30 条防止 AppState 膨胀，同时保证 UI 可响应。

## 与其他系统的交互

- **BashTool**：`spawnShellTask` 把长时 Shell 命令转为 `LocalShellTask`，完成后通知回 LLM
- **AgentTool**：`registerAsyncAgent` 创建 `LocalAgentTask` 或 `RemoteAgentTask`
- **AppState / Store**：所有 TaskState 存 `AppState.tasks`，变更通过 `updateTaskState` 原子更新触发 Ink 重渲染
- **通知系统**：任务完成后调 `enqueuePendingNotification`，消息在 LLM 下一轮前插入 `messages[]`
- **autoDream 服务**：触发 `registerDreamTask`，在 footer 显示 Dream 进度
- **Memory 系统**：DreamTask 执行完后更新 `MEMORY.md` 文件
