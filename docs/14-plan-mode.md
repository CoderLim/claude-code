# Plan 模式

## 是什么 & 解决什么问题

Plan 模式是一种"只读探索 + 设计"的受限执行状态。进入后，LLM **只能**使用只读工具（Glob、Grep、Read）和写计划文件，**不能**执行任何有副作用的操作（写文件、运行命令、提交代码等）。

解决的核心问题：在非平凡任务中，LLM 如果直接开始改代码，往往因为对代码库理解不足而走弯路，或者选择了用户并不认可的实现方案。Plan 模式强制先探索、先对齐，再动手。

工作流程：

```
用户请求 → EnterPlanMode（需用户确认）→ 探索代码库 → 写计划文件
→ ExitPlanMode（需用户审批计划）→ 恢复原有权限 → 开始实现
```

---

## 工作原理

### 权限模式切换

Plan 模式本质上是 `ToolPermissionContext.mode` 字段的一个取值 `'plan'`，与 `'default'`、`'acceptEdits'`、`'bypassPermissions'`、`'auto'` 并列。

进入 plan 模式时：
1. 当前 mode 被保存到 `prePlanMode` 字段（用于退出时恢复）
2. mode 切换为 `'plan'`
3. 若原来是 `auto` 模式，根据 `shouldPlanUseAutoMode()` 决定是否在 plan 期间保持 auto 的分类器活跃

退出 plan 模式时：
1. mode 从 `prePlanMode` 恢复
2. 若原来是 `auto` 且 auto 的 gate 已关闭（circuit breaker），回退到 `'default'`，并弹出通知

### 工具调用限制

Plan 模式**不是**通过过滤工具列表来限制的，而是通过 **permission 检查**在每次工具调用时拦截。

`permissions.ts` 中的关键逻辑（step 2a）：

```ts
const shouldBypassPermissions =
  appState.toolPermissionContext.mode === 'bypassPermissions' ||
  (appState.toolPermissionContext.mode === 'plan' &&
    appState.toolPermissionContext.isBypassPermissionsModeAvailable)
```

- 只有当用户**原来就是** `bypassPermissions` 模式时，plan 模式内才会继承 bypass 行为
- 普通 plan 模式下，所有有副作用的工具调用都会触发 `ask`（弹出权限确认框）

**计划文件是唯一例外**：`filesystem.ts` 中的 `checkEditableInternalPath()` 对当前 session 的计划文件路径返回 `allow`，无需权限确认。

### System Prompt 注入（Attachment 机制）

每次 LLM 调用前，`attachments.ts` 检查当前 mode，若为 `'plan'`，则注入 `plan_mode` attachment，转换为 meta user message 插入对话历史。

注入频率控制（避免 context 膨胀）：
- 第 1 次进入 plan 模式：始终注入完整指令（full reminder）
- 后续每 N 轮注入一次完整版，其余轮次注入简短版（sparse reminder）
- 退出 plan 模式时，注入一次 `plan_mode_exit` attachment（一次性通知）
- 重新进入 plan 模式且计划文件已存在时，注入 `plan_mode_reentry` attachment

---

## 实现细节

### EnterPlanModeTool

**文件**：`src/tools/EnterPlanModeTool/EnterPlanModeTool.ts`

- `shouldDefer: true`：调用此工具时先暂停执行，弹出权限确认框（`EnterPlanModePermissionRequest`）
- `isReadOnly(): true`：工具本身被标记为只读
- `isEnabled()`：当 `--channels` 模式激活时禁用（因为退出时的审批对话框需要终端 UI，channels 模式下无法显示）
- 不接受任何参数（空 schema）
- `call()` 内部调用 `prepareContextForPlanMode()` 更新 `toolPermissionContext`，将 mode 设为 `'plan'`
- `mapToolResultToToolResultBlockParam()` 向 LLM 返回工作流指令，核心内容：**不要写或编辑任何文件（计划文件除外），这是只读探索和规划阶段**

**UI 确认框**：`src/components/permissions/EnterPlanModePermissionRequest/EnterPlanModePermissionRequest.tsx`

选项：
- "Yes, enter plan mode" → 调用 `toolUseConfirm.onAllow()`，permission update 设 `mode: 'plan'`
- "No, start implementing now" → 拒绝，LLM 直接开始实现

### ExitPlanModeV2Tool

**文件**：`src/tools/ExitPlanModeTool/ExitPlanModeV2Tool.ts`

- `shouldDefer: true`：弹出审批对话框
- `isReadOnly(): false`：会写磁盘（同步计划文件内容）
- `requiresUserInteraction()`：非 teammate 场景返回 `true`（需要用户确认）
- `validateInput()`：若当前 mode 不是 `'plan'`，直接返回 error，防止在 plan 模式外误调用
- `call()` 内部：
  1. 若是 teammate 且 `isPlanModeRequired()`，将计划内容通过 mailbox 发送给 team leader，等待审批
  2. 否则，直接恢复 `prePlanMode`，设置 `needsPlanModeExitAttachment` flag（触发下一轮的 `plan_mode_exit` attachment）
- `mapToolResultToToolResultBlockParam()` 向 LLM 返回已批准的计划内容，指示可以开始编码

**退出时的权限选项**（`ExitPlanModePermissionRequest`）：

用户可选择退出后进入哪种模式：
- `yes-accept-edits`：切换到 acceptEdits 模式（自动接受文件编辑）
- `yes-bypass-permissions`：切换到 bypassPermissions 模式（跳过所有权限确认）
- `yes-resume-auto-mode`：恢复 auto 模式（AI 分类器自动判断权限）
- `yes-default-keep-context`：恢复默认模式，保留上下文
- `yes-auto-clear-context`：恢复默认模式，清空上下文（新开一轮对话）
- `ultraplan`：启动 ultraplan（远程 cloud session）
- `no`：留在 plan 模式，继续修改计划

### 计划文件管理

**文件**：`src/utils/plans.ts`

- 计划文件默认存放在 `~/.claude/plans/` 目录（可通过 `settings.json` 的 `plansDirectory` 字段自定义）
- 文件名由随机词组 slug 生成（如 `happy-river.md`），每个 session 唯一，缓存在 `STATE.planSlugCache`
- `/clear` 命令会清空 slug 缓存，确保新 session 使用新计划文件
- 子 agent（`agentId` 存在）有独立的计划文件路径

### AppState 中的相关字段

**文件**：`src/bootstrap/state.ts`

```ts
// ToolPermissionContext 中
mode: PermissionMode          // 当前权限模式，'plan' 表示 plan 模式
prePlanMode?: PermissionMode  // 进入 plan 模式前的 mode，退出时恢复

// STATE 全局状态
hasExitedPlanModeInSession: boolean  // 本 session 是否曾退出过 plan 模式（用于 re-entry 检测）
needsPlanModeExitAttachment: boolean // 是否需要在下一轮注入 plan_mode_exit attachment
planSlugCache: Map<SessionId, string> // session -> 计划文件 slug 缓存
```

### System Prompt 注入内容

**文件**：`src/utils/messages.ts`

Plan 模式的 attachment 转换为 meta user message，包含以下工作流指令（5 阶段标准版）：

1. **Phase 1 - Initial Understanding**：并行启动最多 N 个 `explore` 类型 subagent 探索代码库
2. **Phase 2 - Design**：启动 `plan` 类型 subagent 设计实现方案
3. **Phase 3 - Review**：读取关键文件，确认方案与用户意图对齐，用 `AskUserQuestion` 澄清疑问
4. **Phase 4 - Write Plan File**：将最终方案写入计划文件
5. **Phase 5 - Call ExitPlanMode**：调用 `ExitPlanMode` 请求用户审批

每轮结尾**只能**以 `AskUserQuestion`（澄清需求）或 `ExitPlanMode`（请求审批）结束，不得直接用文字询问"计划是否可以？"

**Interview Phase（实验性）**：通过 feature flag `tengu_plan_mode_interview_phase` 或 `USER_TYPE=ant` 启用，改为迭代式对话工作流——边探索边问用户问题，逐步完善计划文件，无固定的 5 阶段结构。

### UI 层的 Plan 模式状态显示

- **进入确认**：`EnterPlanModePermissionRequest` 组件，颜色主题 `planMode`，标题 "Enter plan mode?"
- **退出审批**：`ExitPlanModePermissionRequest` 组件，展示计划文件内容供用户审阅，提供多种退出后权限模式选项
- **ModelPicker**：plan 模式下选择模型会显示提示 "Currently using X for this session (set by plan mode)"，选择新模型会解除 plan mode 的模型绑定

---

## 边界条件 & 注意事项

**channels 模式下 plan 模式被禁用**

`EnterPlanModeTool.isEnabled()` 和 `ExitPlanModeV2Tool.isEnabled()` 都检查 `getAllowedChannels().length > 0`。原因：退出 plan 模式需要弹出终端 UI 审批对话框，channels（Telegram/Discord 等）模式下用户不在终端旁边，对话框会永久挂起。两个工具配套禁用，防止进入后无法退出的陷阱。

**plan 模式不等于只读模式**

工具列表本身不变，限制来自 permission 检查。用户在审批对话框中点击"允许"仍然可以执行写操作——plan 模式的约束是通过 system prompt 指令告知 LLM"不要这样做"，而非技术上完全阻止。

**与 bypassPermissions 的区别**

| 维度 | plan 模式 | bypassPermissions |
|------|-----------|-------------------|
| 工具调用限制 | 通过 system prompt 指令限制 LLM 行为 | 跳过所有权限确认，LLM 可自由执行 |
| 权限确认框 | 有副作用的工具仍会弹框 | 不弹框，全部自动允许 |
| 计划文件 | 唯一可免确认写入的文件 | 所有文件均可写 |
| 继承关系 | 若原来是 bypass 模式，plan 内继承 bypass 行为 | — |

**prePlanMode 与 auto 模式的交互**

从 auto 模式进入 plan 模式时，`prepareContextForPlanMode()` 根据 `shouldPlanUseAutoMode()` 决定是否在 plan 期间保持 AI 分类器活跃。退出时，若 auto 的 gate 已被 circuit breaker 关闭，则不恢复 auto 而是回退到 `default`，并通过 notification 告知用户。

**Teammate 场景的 plan 审批**

当 plan 模式在 subagent（teammate）中激活且 `isPlanModeRequired()` 为真时，`ExitPlanMode` 不弹本地对话框，而是将计划内容通过 mailbox 发送给 team leader，等待 leader 回复 `plan_approval_response`。

**计划文件的持久性**

计划文件写入磁盘（`~/.claude/plans/<slug>.md`），在 session 之间持久存在。重新进入 plan 模式时若文件已存在，会触发 `plan_mode_reentry` attachment，提示 LLM 先读取历史计划再决定是延续还是重写。
