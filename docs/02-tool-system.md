# 工具系统（Tool System）

## 概述

工具系统是 Claude Code Harness 的执行核心，负责将 LLM 的 tool_use 决策转化为真实的系统操作（文件读写、Shell 命令、网络请求等）。LLM 选择调用哪个工具、传什么参数；Harness 负责执行、检查权限、返回结果。工具系统位于 QueryEngine（外层循环）和具体实现之间，是 Harness 与 LLM 的主要交互接缝之一。

## 核心职责

- 定义所有工具的统一接口（`Tool<Input, Output>` 泛型类型）
- 维护工具注册表，按 feature flag 和环境变量动态组装可用工具集
- 在工具执行前调用 `canUseTool` 进行权限检查
- 把工具执行结果（含进度流）转化为 `messages[]` 追加到对话上下文
- 管理工具的并发安全性（`isConcurrencySafe`）和中断行为（`interruptBehavior`）

## 架构概览

```
LLM（tool_use block）
  ↓
QueryEngine.ts（识别 tool_use，分发执行）
  ↓
canUseTool（useCanUseTool hook，权限检查）
  ↓
tool.call(args, context, canUseTool, parentMessage, onProgress)
  ↓ 并行（isConcurrencySafe = true）或串行
ToolResult<Output>
  ↓
messages[] 追加 tool_result block
  ↓ 下一轮 LLM 请求
```

工具注册入口：`src/tools.ts` → `getAllBaseTools()` → 返回 `Tools`（`Tool[]`）数组。运行时通过 `getTools(permissionContext)` 过滤掉 deny 规则匹配的工具后，注入到 `ToolUseContext.options.tools`。

## 核心实现

### Tool 接口（基类）

**文件**：`src/Tool.ts`

每个工具是一个满足 `Tool<Input, Output, Progress>` 接口的对象（非 class，duck typing）：

| 字段/方法 | 说明 |
|---|---|
| `name: string` | 工具名，LLM 通过此名调用 |
| `aliases?: string[]` | 旧名别名，向后兼容 |
| `inputSchema: z.ZodType` | 入参 Zod Schema，用于 LLM tool schema 生成和运行时校验 |
| `call(args, context, canUseTool, parentMessage, onProgress)` | 工具执行主函数，返回 `Promise<ToolResult<Output>>` |
| `description(input, options)` | 生成 UI 展示文本（异步，可依赖输入） |
| `isEnabled()` | 是否在当前环境可用 |
| `isConcurrencySafe(input)` | 是否可与其他工具并行执行 |
| `isReadOnly(input)` | 是否只读（影响权限分类） |
| `isDestructive?(input)` | 是否不可逆操作（删除、覆盖、发送） |
| `interruptBehavior?()` | 用户插入新消息时：`'cancel'` 停止 / `'block'` 继续等待 |
| `maxResultSizeChars` | 结果超限时序列化到磁盘，Claude 收到预览+文件路径 |
| `shouldDefer?: boolean` | 设为 true 则工具延迟加载，需先通过 ToolSearch 发现 |
| `alwaysLoad?: boolean` | 总是出现在初始 prompt，不受 ToolSearch 影响 |
| `searchHint?: string` | ToolSearch 关键词匹配提示（3-10 词） |

`ToolUseContext` 是工具执行的完整上下文，包含：`options`（commands/tools/model 等配置）、`abortController`、`getAppState()`/`setAppState()`、`setToolJSX`（UI 注入）、`messages`、`readFileState`（文件缓存）等。

### 工具注册表

**文件**：`src/tools.ts`

`getAllBaseTools()` 是工具注册的唯一真值来源，按条件组装：

**无条件加载的核心工具**（约 20 个）：
- `AgentTool`、`BashTool`、`FileReadTool`、`FileEditTool`、`FileWriteTool`
- `GlobTool`、`GrepTool`（若有嵌入式搜索工具则跳过）
- `WebFetchTool`、`WebSearchTool`、`NotebookEditTool`
- `TodoWriteTool`、`TaskStopTool`、`AskUserQuestionTool`、`SkillTool`
- `EnterPlanModeTool`、`ExitPlanModeV2Tool`、`BriefTool`
- `ListMcpResourcesTool`、`ReadMcpResourceTool`

**feature flag 条件加载**：

| Feature Flag | 工具 |
|---|---|
| `USER_TYPE === 'ant'` | `REPLTool`、`ConfigTool`、`TungstenTool`、`SuggestBackgroundPRTool` |
| `PROACTIVE` / `KAIROS` | `SleepTool`、`SendUserFileTool`、`PushNotificationTool` |
| `AGENT_TRIGGERS` | `CronCreateTool`、`CronDeleteTool`、`CronListTool` |
| `AGENT_TRIGGERS_REMOTE` | `RemoteTriggerTool` |
| `MONITOR_TOOL` | `MonitorTool` |
| `KAIROS_GITHUB_WEBHOOKS` | `SubscribePRTool` |
| `WEB_BROWSER_TOOL` | `WebBrowserTool` |
| `UDS_INBOX` | `ListPeersTool` |
| `WORKFLOW_SCRIPTS` | `WorkflowTool` |
| `CONTEXT_COLLAPSE` | `CtxInspectTool` |
| `TERMINAL_PANEL` | `TerminalCaptureTool` |
| `HISTORY_SNIP` | `SnipTool` |
| `isAgentSwarmsEnabled()` | `TeamCreateTool`、`TeamDeleteTool` |
| `isWorktreeModeEnabled()` | `EnterWorktreeTool`、`ExitWorktreeTool` |
| `isTodoV2Enabled()` | `TaskCreateTool`、`TaskGetTool`、`TaskUpdateTool`、`TaskListTool` |
| `CLAUDE_CODE_VERIFY_PLAN` | `VerifyPlanExecutionTool` |
| `isPowerShellToolEnabled()` | `PowerShellTool` |
| `isToolSearchEnabledOptimistic()` | `ToolSearchTool` |

**懒加载（解决循环依赖）**：`TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool` 使用 getter 函数延迟 require。

### 工具按类别分组

| 类别 | 工具 |
|---|---|
| **Shell 执行** | `BashTool`、`PowerShellTool`、`REPLTool` |
| **文件操作** | `FileReadTool`、`FileEditTool`、`FileWriteTool`、`GlobTool`、`GrepTool`、`NotebookEditTool` |
| **网络** | `WebFetchTool`、`WebSearchTool`、`WebBrowserTool` |
| **Agent 与多智能体** | `AgentTool`、`TeamCreateTool`、`TeamDeleteTool`、`SendMessageTool`、`ListPeersTool` |
| **任务管理** | `TaskCreateTool`、`TaskGetTool`、`TaskUpdateTool`、`TaskListTool`、`TaskStopTool`、`TaskOutputTool` |
| **Plan 模式** | `EnterPlanModeTool`、`ExitPlanModeV2Tool` |
| **定时任务** | `CronCreateTool`、`CronDeleteTool`、`CronListTool`、`RemoteTriggerTool` |
| **MCP 集成** | `MCPTool`、`ListMcpResourcesTool`、`ReadMcpResourceTool`、`McpAuthTool` |
| **交互与通知** | `AskUserQuestionTool`、`PushNotificationTool`、`SleepTool` |
| **代码辅助** | `LSPTool`、`SkillTool`、`WorkflowTool` |
| **Worktree** | `EnterWorktreeTool`、`ExitWorktreeTool` |
| **其他内部工具** | `ConfigTool`、`BriefTool`、`TodoWriteTool`、`ToolSearchTool`、`SyntheticOutputTool`、`TungstenTool`、`MonitorTool`、`SnipTool`、`SendUserFileTool`、`SubscribePRTool`、`CtxInspectTool`、`TerminalCaptureTool`、`VerifyPlanExecutionTool` |

### 权限检查时机

**文件**：`src/hooks/useCanUseTool.tsx`

权限检查发生在 `tool.call()` 被调用之前，由 QueryEngine 将 `canUseTool` 函数注入给每个工具。工具在执行实际操作前可自行调用 `canUseTool(input)` 检查。

权限检查链：
1. `alwaysDenyRules` — 无条件拒绝
2. hooks（PreToolUse）— shell 命令钩子，可 auto-approve / auto-deny
3. `alwaysAllowRules` — 白名单自动通过
4. 分类判断（`isReadOnly`、`isDestructive` 等）→ 弹 UI 确认或自动通过

### 工具调用完整流程

```
QueryEngine 收到 LLM 的 tool_use block
  → findToolByName(tools, toolName) 查找工具实例
  → 并行或串行执行（isConcurrencySafe）
      → canUseTool(input, context) 权限检查
          → 若 deny → 返回 "permission denied" tool_result
          → 若 allow → tool.call(args, context, canUseTool, parentMessage, onProgress)
              → onProgress 回调推送流式进度（UI 实时显示）
              → 返回 ToolResult<Output>
  → 将结果组装为 tool_result block 追加到 messages[]
  → 进入下一轮 LLM 请求
```

### BashTool 特殊机制

**文件**：`src/tools/BashTool/BashTool.tsx`

- **后台执行**：阻塞 Bash 超过 `ASSISTANT_BLOCKING_BUDGET_MS`（15s）后自动后台化，使用 `spawnShellTask` 转为 `LocalShellTask`
- **输出写文件**：stdout 直接写文件 fd（绕过 JS 流），周期性轮询文件尾部（1s 间隔）
- **沙箱**：`shouldUseSandbox()` 决定是否用 macOS sandbox-exec 隔离
- **sleep 拦截**：`MONITOR_TOOL` 开启后拦截 `sleep N` 命令并报错

### AgentTool 特殊机制

**文件**：`src/tools/AgentTool/AgentTool.tsx`

- 启动子 Agent（`runAgent`），可选本地（`LocalAgentTask`）或远程（`RemoteAgentTask`）
- 支持 fork 模式（`FORK_AGENT`）：子 Agent 共享父级 context，但有 copy-on-write 隔离
- 协调者模式（`COORDINATOR_MODE`）下 Agent 只见任务分配相关工具

## 关键设计决策

1. **Duck typing 而非 class 继承**：`Tool` 是纯 TypeScript interface，工具以对象字面量实现，避免类层次复杂性，便于 tree-shaking 和测试 mock。

2. **Feature flag 动态组装，而非运行时过滤**：工具在 `getAllBaseTools()` 时就按 flag 决定是否放入数组，而不是全量加载后过滤，降低了 bundle size（Bun dead code elimination）。

3. **`maxResultSizeChars` + 磁盘序列化**：工具结果超限时写磁盘、给 Claude 文件路径，而非截断。保持了结果完整性，同时控制 context window 用量。

4. **权限检查注入工具**：`canUseTool` 以函数参数形式注入 `tool.call()`，而非全局状态，使工具在嵌套 Agent 场景中可以持有不同的权限上下文。

## 与其他系统的交互

- **QueryEngine**：外层消费者，调用 `getTools()` 拿到工具列表，驱动 `tool.call()` 执行，将结果注入 `messages[]`
- **useCanUseTool（Hook 系统）**：权限检查的实际执行者，工具在调用前必须通过
- **Task 系统**：`BashTool` 可将长时命令转 `LocalShellTask`；`AgentTool` 创建 `LocalAgentTask`/`RemoteAgentTask`
- **AppState / Store**：`ToolUseContext` 通过 `getAppState()`/`setAppState()` 读写全局状态（任务列表、文件缓存等）
- **Skills 系统**：`SkillTool` 加载并执行 Markdown 指令文件（skills）
- **MCP 系统**：`MCPTool` 是动态注册的，每个 MCP server connection 生成一批 `Tool` 实例追加到工具集
