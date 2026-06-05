# 上下文压缩（Context Compression）

## 是什么 & 解决什么问题

LLM 的上下文窗口有硬性上限。随着对话轮次增加、工具调用累积，`messages[]` 会不断增长，最终触碰 token 限制导致请求失败。Claude Code 实现了**5层渐进式压缩机制**，在每轮 API 请求前按需收缩上下文，越靠后越激进，能用细粒度解决就不做全量摘要。

**触发时机**：每轮 `queryLoop` 迭代开头，组装 `messagesForQuery` 后，进入 `query()` 循环之前。

---

## 5层压缩机制

### 第1层：snip compact

**文件**：feature gate `HISTORY_SNIP`

把历史中**过长的 tool_result** 截断为 `[snipped]` 占位符，保留消息结构但移除实际内容。

- 只截内容，不删消息——LLM 仍能看到"这里有个工具调用结果"的骨架
- 适用于：大量文件读取结果、长 Bash 输出等一次性数据，后续轮次不再需要其完整内容
- 最保守的一层，信息损失最小

### 第2层：microcompact

**文件**：`src/context/microcompact.ts`

Strip 早期轮次中**已过时的 tool_result 内容**，同时通知 `cache break detector`（因为消息内容改变，缓存断点需要重新计算）。

- 比 snip 更激进：直接移除内容，而非截断
- **延迟生效**：cached microcompact 的结果延迟到当次 API 响应完成后才写回 `messages[]`，避免正在进行的响应读到不一致的上下文
- 目标：把不再有用的中间过程数据清出去，保留对话的语义连贯性

### 第3层：context collapse

**文件**：`src/context/contextCollapse.ts`

把**已完成的子对话片段**（如一个完整的 Agent 子任务）折叠成摘要消息。

- 细粒度折叠：只折叠边界清晰的片段，不动整个对话历史
- 故意排在 autocompact 之前：能细粒度解决就不做全量压缩
- 413（`prompt_too_long`）错误恢复时**优先**尝试 context collapse drain（逐步折叠），再退化到 reactive compact

### 第4层：autocompact

**文件**：`src/context/autocompact.ts`

当 token 用量接近阈值时，触发**全量对话摘要**：调用 LLM 将整个 `messages[]` 压缩成一段结构化摘要，替换原始消息列表。

```
触发条件：token 用量 >= autocompact 阈值（由 GrowthBook 配置）
执行：LLM sideQuery → 生成摘要 → messagesForQuery 替换为 [摘要消息]
副作用：写入 compact_boundary 标记（持久化层记录压缩点）
```

**未启用时的回退**：若 autocompact 未启用且 token 超过硬性上限，`queryLoop` 直接 yield 错误并返回 `blocking_limit`，对话终止。

**`autoCompactTracking`**：`AppState` 中跟踪压缩状态，用于 UI 进度展示和 telemetry。

### 第5层：reactive compact

**文件**：`src/context/reactiveCompact.ts`

**被动触发**——只在收到 `prompt_too_long`（413）错误时启动，是最后的兜底手段。

```
API 返回 413
  ↓ 先尝试 context collapse drain（逐步折叠）
  ↓ 仍失败 → reactive compact（紧急全量压缩）
  ↓ 仍失败 → surface 错误给用户
```

- 与 autocompact 的区别：autocompact 是**预防性**的（提前压缩），reactive compact 是**救火性**的（已经 413 了才触发）
- 产生 `compact_boundary` 标记，支持增量恢复（`--resume` 时从 boundary 点重建上下文）

---

## 错误暂扣（Withhold）机制

`max_output_tokens` 和 `prompt_too_long`（413）这两类错误在流式阶段**不立即 yield** 给调用方，而是暂扣在内部，等待恢复逻辑：

```
流式收到 413
  → 暂扣错误，不 yield
  → 尝试 context collapse drain
  → 仍失败 → reactive compact
  → 仍失败 → yield 错误给用户
```

**为什么要暂扣**：若 SDK 调用方提前收到 error 消息，会终止会话，导致恢复逻辑根本无法运行。暂扣让 harness 有机会自愈。

`max_output_tokens` 的恢复路径不同：先尝试将 `max_tokens` escalate 到 64k，再注入 meta 消息 "Output token limit hit. Resume directly…" 最多重试 3 次（`MAX_OUTPUT_TOKENS_RECOVERY_LIMIT`）。

---

## compact_boundary — 压缩点标记

**文件**：`src/state/AppStateStore.ts`

每次执行 autocompact 或 reactive compact 后，写入一条 `compact_boundary` 消息到 transcript。

作用：
- `--resume` 时从 boundary 点重建上下文，无需重放完整历史
- UI 可在时间线上标记"此处发生过压缩"
- telemetry 追踪压缩频率和触发原因

---

## 整体触发顺序

```
每轮 queryLoop 迭代
  ↓
snipCompactIfNeeded()       第1层：截断过长 tool_result
  ↓
microcompact()              第2层：移除过时 tool_result 内容
  ↓
contextCollapse()           第3层：折叠已完成子对话片段
  ↓
autocompact()               第4层：全量摘要（阈值预警时）
  ↓
进入 query() 循环
  ↓ (API 返回 413)
reactive compact            第5层：救火性紧急压缩
```

越靠后越激进，越靠后信息损失越大。设计原则：**能用细粒度解决就不做全量摘要**。
