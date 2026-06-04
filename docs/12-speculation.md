# Speculation（推测执行专题）

## 是什么 & 解决什么问题

Speculation（推测执行）是 Claude Code 的性能优化机制：在用户还在打字时，后台**预先执行** LLM 对用户可能提交的下一条 prompt 的响应，包括工具调用（文件读取、搜索等）。用户实际提交时，如果输入内容与预测吻合，直接采用已预执行的结果，跳过 API 等待时间，感觉上响应是即时的。

**启用条件**：`process.env.USER_TYPE === 'ant'`（仅 Anthropic 内部员工）且 `globalConfig.speculationEnabled !== false`。

## 工作原理

### 整体流程

```
上一轮 Claude 响应结束
  ↓
PromptSuggestion 生成预测文本（generateSuggestion → Sonnet sideQuery）
  ↓
startSpeculation(suggestionText, context)
  ↓
runForkedAgent({
  promptMessages: [suggestionText],
  cacheSafeParams: createCacheSafeParams(context),  // 共享父级缓存
  skipTranscript: true,
  skipCacheWrite: true (fire-and-forget),
  canUseTool: speculative_permission_check
})

用户打字中：
  ↓ 输入与预测不符 → abortSpeculation() → safeRemoveOverlay() [丢弃]
  ↓ 输入与预测匹配 → acceptSpeculation()
      → copyOverlayToMain()  [overlay 文件复制到真实目录]
      → prepareMessagesForInjection(speculatedMessages)
        → 过滤未完成/中断的 tool_use 块
      → speculation messages 注入主 messages[]
      → boundary === 'complete' → 不再发 API 请求（直接显示结果）
      → boundary ≠ 'complete' → 从 boundary 点继续发一次请求

speculation 完成后（boundary = complete 时）：
  → generatePipelinedSuggestion()  [立即生成下一条预测，形成链式预执行]
```

### 文件 Overlay（Copy-on-Write 隔离）

**路径**：`~/.claude/tmp/speculation/{pid}/{id}/`

推测执行期间的文件写操作不落到真实文件系统，而是写入 overlay 目录：

**Copy-on-Write 步骤**：
1. 首次写入 `{cwd}/foo.ts` 前，先 `copyFile({cwd}/foo.ts, {overlay}/foo.ts)`（保留原文件快照）
2. 写操作重定向到 `{overlay}/foo.ts`
3. 后续读操作：若该相对路径已在 `writtenPathsRef.current`，也重定向到 overlay（读到的是推测版本）
4. 用户接受：`copyOverlayToMain()` 批量将 overlay 文件覆盖到 `{cwd}/`
5. 用户拒绝：`safeRemoveOverlay()` 删除整个 overlay 目录

**`writtenPathsRef.current`**：`Set<string>`，存储已写入的相对路径（相对 cwd）。保持为 mutable ref 而非 React state，避免每次写操作触发重渲染。

**安全边界**：overlay 外路径（`relative(cwd, filePath)` 产生 `isAbsolute` 或 `..` 开头的路径）的写操作直接 deny；读操作允许（不重定向，直接读真实文件）。

## 实现细节

### 工具许可表

**文件**：`src/services/PromptSuggestion/speculation.ts`

`canUseTool` 是 speculation 的核心：

| 工具类型 | 处理方式 | 说明 |
|---|---|---|
| `Read`、`Glob`、`Grep`、`ToolSearch`、`LSP`、`TaskGet`、`TaskList` | **允许**（可能重定向到 overlay）| 只读工具，安全 |
| `Edit`、`Write`、`NotebookEdit`（mode = acceptEdits/bypassPermissions/plan+bypass）| **允许**（写入 overlay）| 权限充足时才允许 |
| `Edit`、`Write`、`NotebookEdit`（其他 mode）| **停止**，记 `edit` boundary | 权限不足，不执行 |
| `Bash`（只读命令，非 `cd`）| **允许** | `checkReadOnlyConstraints` 通过 |
| `Bash`（写操作或 `cd`）| **停止**，记 `bash` boundary | 不可逆操作，停下 |
| 其他工具 | **deny**，记 `denied_tool` boundary | 网络、MCP、Agent 等 |

**最大限制**：`MAX_SPECULATION_TURNS = 20`，`MAX_SPECULATION_MESSAGES = 100`。超限时 `abortController.abort()`，speculation 中止。

### 4 种 CompletionBoundary 类型

**文件**：`src/state/AppStateStore.ts`

```ts
type CompletionBoundary =
  | { type: 'complete'; completedAt: number; outputTokens: number }
    // speculation 正常完成（Claude 停止生成）
  | { type: 'bash'; command: string; completedAt: number }
    // 遇到写/cd Bash 命令停下
  | { type: 'edit'; toolName: string; filePath: string; completedAt: number }
    // 遇到文件写操作停下（权限不足时）
  | { type: 'denied_tool'; toolName: string; detail: string; completedAt: number }
    // 遇到不允许的工具停下
```

**接受时行为**：
- `boundary.type === 'complete'`：speculation 跑完了，直接用结果，**不再发 API 请求**
- `boundary.type !== 'complete'`：从 boundary 点继续，发一次 API 请求（用户看到短暂的"继续执行"）

### AppState 中的 Speculation 状态

**文件**：`src/state/AppStateStore.ts`

```ts
type SpeculationState =
  | { status: 'idle' }
  | {
      status: 'active'
      id: string                         // UUID 前 8 位，对应 overlay 目录名
      abort: () => void                  // 调用即取消
      startTime: number
      messagesRef: { current: Message[] }     // 可变 ref，accumulate 中间消息
      writtenPathsRef: { current: Set<string> } // 已写 overlay 的相对路径
      boundary: CompletionBoundary | null       // null = 还在跑
      suggestionLength: number
      toolUseCount: number
      isPipelined: boolean              // 是否来自 pipelining 链
      pipelinedSuggestion?: {           // 下一条预测（pipeline 生成的）
        text: string
        promptId: 'user_intent' | 'stated_intent'
        generationRequestId: string | null
      } | null
    }
```

**为什么 `messagesRef` 是 mutable ref 而非 state**：`onMessage` 回调每条消息都调用，每次追加都触发 React re-render 会有性能问题。mutable ref 直接 push，只在 `toolUseCount` 变化时更新 AppState（触发进度 UI 更新）。

### prepareMessagesForInjection — 消息清理

接受 speculation 时，不能直接把所有 speculation 消息注入主线程，需要过滤：

- **去除 `thinking` 和 `redacted_thinking` block**：扩展思维不注入主消息
- **去除未完成的 `tool_use` block**：boundary 点处被中止的工具调用没有对应结果，LLM 会报错
- **去除对应的失败/中断 `tool_result` block**
- **去除独立的中断消息**：`INTERRUPT_MESSAGE` / `INTERRUPT_MESSAGE_FOR_TOOL_USE` 是 speculation 内部的中止标记，不应出现在主对话中
- **去除全空白内容的消息**：API 要求 text content block 必须含非空白字符

### Pipelining — 连续预执行链

**文件**：`src/services/PromptSuggestion/speculation.ts` → `generatePipelinedSuggestion()`

当 boundary = `complete`（speculation 跑完）后，**立即**调用 `generatePipelinedSuggestion()`，基于"已提交的 prompt + speculation 的结果消息"作为新的上下文，生成下一条可能的 prompt 预测，并存入 `AppState.speculation.pipelinedSuggestion`。

用户接受当前 speculation 后，若 `pipelinedSuggestion` 已就绪，立即用它启动下一轮 speculation，形成链式预执行：

```
[上一轮结束] → Suggestion A → startSpeculation(A) → [A 完成] → Suggestion B（pipeline）
[用户接受 A] → 立即 startSpeculation(B, isPipelined=true)
[用户接受 B] → 立即 startSpeculation(C, isPipelined=true) → ...
```

若用户接受时 `pipelinedSuggestion` 还未就绪（Sonnet 还在生成），则退化为正常的 PromptSuggestion 流程（先等生成，再 speculation）。

## 边界条件 & 注意事项

1. **任何背景任务状态变化都 abort speculation**：`enqueueShellNotification` 中的 `abortSpeculation(setAppState)` 确保任务完成通知的状态变化不被陈旧的 speculation 结果覆盖。

2. **`cwd` 变化（`cd` 命令）后 speculation 停下**：Bash 工具的 `commandHasAnyCd` 检测后记 `bash` boundary 停止，防止 overlay 路径计算在 cwd 变化后错乱。

3. **写入 overlay 外的路径被 deny**：`isAbsolute(rel) || rel.startsWith('..')` 判断，绝对路径和路径穿越均被拒绝，防止 speculation 修改 cwd 范围外的系统文件。

4. **`requireCanUseTool: true`**：speculation 的 `runForkedAgent` 强制设置此标志，确保 `canUseTool` 在每次工具调用前都被调用（正常子 agent 可能在 hooks auto-approve 时跳过），overlay 路径重写依赖于此。

5. **cache 不污染主线程**：`skipCacheWrite: true` + `skipTranscript: true`，speculation 是 fire-and-forget fork，不产生新的 cache 条目，不留旁链 transcript。

6. **ant-only 反馈消息**：`createSpeculationFeedbackMessage` 对 `USER_TYPE === 'ant'` 用户插入 `[ANT-ONLY] Speculated N tool uses · +XXms saved` 系统消息，供内部观察 speculation 效果。
