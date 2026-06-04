# Prompt Cache（提示词缓存专题）

## 是什么 & 解决什么问题

Prompt Cache 是 Anthropic 服务端的 KV 缓存复用机制：如果相邻两次 API 请求的 messages 前缀完全相同，服务端跳过对这段前缀的 Attention 计算，直接复用已缓存的 KV 矩阵，显著降低延迟和成本（缓存命中的 tokens 按更低价格计费）。

对于 Claude Code，每轮对话都在原有 messages[] 末尾追加内容，前缀天然共享——所以对话越长，缓存收益越大。主要解决的问题：避免每轮都重新计算庞大 system prompt（tools 列表、CLAUDE.md、memory 等）的 Attention。

## 工作原理

### Attention KV 是什么

Transformer 的每个 token 生成 Query/Key/Value 三个向量：

```
Attention(Q, K, V) = softmax(Q·Kᵀ / √d) · V
```

**KV Cache（单请求内）**：自回归生成时，已计算的 K、V 矩阵按位置缓存，生成第 N+1 个 token 时直接读前 N 个位置的 KV，不重算。

**Prompt Cache（跨请求）**：服务端序列化并持久化 KV 矩阵，下次请求的前缀与已缓存前缀逐 token 对比，命中则跳过该前缀的推理。**前缀任一 token 不同，从该位置往后的全部 KV 失效**。

### cache_control 标记

客户端通过 `cache_control` 字段指示服务端在哪个位置写入缓存：

```ts
getCacheControl() → { type: 'ephemeral', ttl?: '1h', scope?: 'global' }
```

**每次请求只放 1 个 `cache_control` 标记**（设计约束）：多标记会导致服务端保护中间位置的 KV 页，阻止其在下一轮被正常释放，浪费内存。

## 实现细节

### TTL 两档

**文件**：`src/services/api/claude.ts` → `should1hCacheTTL()`

| TTL | 适用条件 |
|---|---|
| 默认 5 分钟 | 所有用户的默认值 |
| 1 小时 | Anthropic 内部员工（`USER_TYPE === 'ant'`）或未超量的 claude.ai 订阅用户 |

1h TTL 由 GrowthBook `tengu_prompt_cache_1h_config` 控制，包含两层门控：
1. **用户资格（`userEligible`）**：在 session 启动时通过 `setPromptCache1hEligible(userEligible)` **锁定**到 bootstrap state，后续 overage 状态变化不再影响此值
2. **querySource 白名单**：`tengu_prompt_cache_1h_config` 配置 `allowlist` 字段，支持 `*` 通配符前缀（如 `"repl_main_thread*"` 匹配所有主线程来源）

**为什么 session 锁定**：如果 mid-session overage 状态翻转导致 TTL 从 1h 降为 5m，`cache_control` 变化会让服务端认为是新的缓存前缀，破坏整个 session 已积累的 ~20K tokens 缓存。锁定防止了这种"缓存炸弹"。

Bedrock 第三方用户可通过 `ENABLE_PROMPT_CACHING_1H_BEDROCK` env var 开启 1h TTL（自行承担费用）。

### addCacheBreakpoints — 标记放置逻辑

**文件**：`src/services/api/claude.ts` → `addCacheBreakpoints()`

```ts
function addCacheBreakpoints(
  messages: Message[],
  enablePromptCaching: boolean,
  querySource?: QuerySource,
  useCachedMC = false,
  newCacheEdits?: CachedMCEditsBlock | null,
  pinnedEdits?: CachedMCPinnedEdits[],
  skipCacheWrite = false,
): MessageParam[]
```

**核心逻辑**：
```ts
// 正常请求：标记放在最后一条消息（新增缓存前缀）
// skipCacheWrite（fire-and-forget fork）：放倒数第二条（不污染主线程缓存前缀）
const markerIndex = skipCacheWrite ? messages.length - 2 : messages.length - 1
```

为何 `skipCacheWrite` 放倒数第二条：倒数第二条是当前已共享的最后一个稳定点（两者都有的最后公共前缀）。在这里设 `cache_control` 是"只读"操作——服务端发现该前缀已存在，本次写入被合并（no-op）；而最后一条消息是 fork 独有的内容，不标记则不产生新缓存条目，不污染主线程将来要继续写的缓存前缀。

### CacheSafeParams — fork agent 缓存复用

**文件**：`src/utils/forkedAgent.ts`

```ts
type CacheSafeParams = {
  systemPrompt: SystemPrompt      // 必须与父请求字节完全相同
  userContext: { [k: string]: string }
  systemContext: { [k: string]: string }
  toolUseContext: ToolUseContext  // 含 tools 列表、thinkingConfig
  forkContextMessages: Message[]  // 父上下文消息（构成公共前缀）
}
```

**机制**：`handleStopHooks`（每轮结束后）调用 `saveCacheSafeParams(params)` 将本轮的参数冻结为 `lastCacheSafeParams`。Fork agent（`/btw`、speculation、promptSuggestion、postTurnSummary 等）调用 `getLastCacheSafeParams()` 获取完全相同的参数，保证 API 请求前缀与父请求一致 → 必然命中缓存。

**`maxOutputTokens` 警告**：如果 fork 设置了 `maxOutputTokens`，claude.ts 会用它 clamp `budget_tokens`（thinking 预算），导致 thinking config 不同于父请求，破坏缓存命中。文档注释明确警告：只在不需要共享缓存时（如 compact summary）才设置此参数。

### skipCacheWrite 语义

**文件**：`src/utils/forkedAgent.ts` → `ForkedAgentParams.skipCacheWrite`

- `skipCacheWrite = true`：`addCacheBreakpoints` 把标记放倒数第二条（不产生新缓存条目）
- 适用场景：fire-and-forget fork（`/btw`、speculation）——它们不会有后续请求来读自己的缓存，留下缓存条目只是浪费服务端内存 + 可能污染主线程前缀
- `skipCacheWrite = false`（默认）：正常写入新缓存条目，供下一轮读取

### Cache Break 检测（两阶段）

**文件**：`src/services/api/promptCacheBreakDetection.ts`

**Phase 1（发请求前）**：`recordPromptState()` 对以下内容哈希，与上轮比对，记录 `pendingChanges`：
- system prompt（`systemHash`，去除 cache_control 后再哈希）
- `cache_control` 本身（`cacheControlHash`，含 scope/TTL 翻转检测）
- tool schemas（`toolsHash` + `perToolHashes`，按工具名精细哈希）
- 工具列表变化（`toolNames`，检测增减）
- model（`model`）
- fast mode（`fastMode`）
- global cache strategy（`tool_based` / `system_prompt` / `none`）
- betas（`betas`，排序后比较）
- effort 值（`effortValue`）
- extra body params（`extraBodyHash`）

**Phase 2（收响应后）**：`checkResponseForCacheBreak()` 比对 `cache_read_input_tokens`：
- 若本次 cache 读取量比上轮**降低超过 5% 且超过 2000 tokens** → 认定为 cache break
- 结合 Phase 1 的 `pendingChanges` 定位原因

**调试输出**：检测到 cache break 时，写 unified diff 到 `~/.claude/tmp/cache-break-{xxxx}.diff`，包含 system prompt、tools schema 的前后变化。

**跟踪的已知非 break 场景**（标记为 "should NOT break anymore"）：
- `autoModeActive`：AFK_MODE_BETA_HEADER sticky-on latched，不再触发 break
- `isUsingOverage`：eligibility 已 session 锁定，不再触发 break
- `cachedMCEnabled`：cached microcompact beta 已 sticky-on latched

## 边界条件 & 注意事项

1. **前缀任一 token 不同，全部失效**：`cache_control` 放置位置以及其上游所有内容（tools 顺序、system prompt 字节）必须稳定。`getAllBaseTools()` 有注释要求与 GrowthBook `claude_code_global_system_caching` 配置保持同步。

2. **structured content（如图片/文件）不可缓存**：API 的 prompt cache 只支持 text block，包含 base64 图片的 messages 在服务端跳过缓存写入。

3. **CacheSafeParams 中 `toolUseContext` 含工具列表**：如果 fork 时 MCP server 连接状态与父请求不同（新 server 刚连接），工具列表不同 → cache miss。`getLastCacheSafeParams()` 的"冻结"特性就是为了隔离这种 race condition。

4. **budget_tokens（thinking）是缓存 key 的一部分**：不同的 thinking 预算 = 不同缓存条目。跨轮次改变 effort 会炸缓存。effort 值也纳入 `pendingChanges` 检测。

5. **global scope**：`shouldUseGlobalCacheScope()` 为 true 时，`getCacheControl` 带 `scope: 'global'`，允许跨用户/组织复用同一 system prompt 的 KV 缓存（适合固定 system prompt 的高吞吐场景）。
