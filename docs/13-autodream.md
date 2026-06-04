# AutoDream — 自动记忆整合

## 是什么 & 解决什么问题

AutoDream 是一个在对话进行中后台自动触发的记忆整合 Agent。它解决的问题是：随着使用时间增长，记忆文件会逐渐碎片化、出现重复或过时内容，而人工整理成本高。AutoDream 在满足时间和会话数量阈值后，悄悄在后台 fork 一个子 Agent，让它执行 `/dream` prompt（4 阶段整合流程），把近期会话的新信息合并到 `MEMORY.md` 等记忆文件中，完成后在主 transcript 底部追加一行 "Improved X files" 的提示。

## 工作原理

触发条件（三关卡，从最廉价到最贵依次检查）：

1. **时间关卡**：距上次整合 >= `minHours`（默认 24 小时）。代价：一次 `stat`。
2. **会话关卡**：`lastConsolidatedAt` 之后有 >= `minSessions`（默认 5 个）新会话（排除当前会话）。代价：扫描 transcript 目录 + 并行 `stat`。
3. **锁关卡**：无其他进程正在整合（CAS 写 lock 文件）。

三关卡全部通过后，fork 子 Agent 异步执行，主对话不阻塞。

**扫描节流**：时间关卡通过但会话关卡未通过时，lock 的 mtime 不前进，导致时间关卡每轮都会通过。为避免每轮都扫描 transcript 目录，用 `SESSION_SCAN_INTERVAL_MS = 10 分钟` 节流。

## 实现细节

### 配置与开关

**文件**：`src/services/autoDream/config.ts`

- 开关优先级：`settings.json` 中的 `autoDreamEnabled` 字段 > GrowthBook feature flag `tengu_onyx_plover.enabled`
- 阈值（`minHours` / `minSessions`）来自 GrowthBook `tengu_onyx_plover`，字段级防御校验，类型错误时回退到默认值（24h / 5 sessions）
- 以下情况强制跳过：KAIROS 模式、Remote 模式、AutoMemory 未启用

### consolidationLock — 锁 & 时间戳

**文件**：`src/services/autoDream/consolidationLock.ts`

Lock 文件路径：`{autoMemPath}/.consolidate-lock`，文件 mtime 即 `lastConsolidatedAt`，文件内容是持有者的 PID。

关键函数：

| 函数 | 作用 |
|------|------|
| `readLastConsolidatedAt()` | 读 lock 文件 mtime，不存在返回 0 |
| `tryAcquireConsolidationLock()` | CAS 写 PID，返回 pre-acquire mtime 或 null（被占用） |
| `rollbackConsolidationLock(priorMtime)` | fork 失败时回退 mtime，priorMtime=0 则直接删文件 |
| `listSessionsTouchedSince(sinceMs)` | 扫描 transcript 目录，返回 mtime > sinceMs 的 sessionId 列表 |
| `recordConsolidation()` | 手动 `/dream` 时乐观写入时间戳（无完成回调，尽力而为） |

**死锁防护**：lock 超过 `HOLDER_STALE_MS = 1 小时`且 PID 已死，下一个进程可强制抢占。

**回滚路径**：fork 失败或用户 kill 时都调用 `rollbackConsolidationLock`，让时间关卡在下次扫描节流过后重新触发。

### autoDream.ts — 主调度逻辑

**文件**：`src/services/autoDream/autoDream.ts`

- `initAutoDream()`：启动时调用一次（在 `backgroundHousekeeping.ts` 中），创建闭包状态（`lastSessionScanAt`），返回 runner 函数
- `executeAutoDream(context, appendSystemMessage)`：每轮对话结束后由 `stopHooks.ts` 调用（仅主会话，`!toolUseContext.agentId`），实际执行 runner
- fork 子 Agent 使用 `runForkedAgent`，`skipTranscript: true`（不写入 transcript），`canUseTool` 限制为只读 Bash 命令
- 完成后若有文件被修改，调用 `appendSystemMessage` 在主 transcript 追加 "Improved N files" 提示

**Tool 限制**：子 Agent 的 Bash 只允许 `ls/find/grep/cat/stat/wc/head/tail` 等只读命令，写操作会被拒绝。

### DreamTask — UI 进度追踪

**文件**：`src/tasks/DreamTask/DreamTask.ts`

DreamTask 是纯 UI 层，把原本不可见的 fork Agent 暴露到 footer pill 和 Shift+↓ 对话框中。

状态结构 `DreamTaskState`：

| 字段 | 含义 |
|------|------|
| `phase` | `'starting'` → `'updating'`（首次 Edit/Write tool_use 到来时翻转） |
| `sessionsReviewing` | 本次整合覆盖的会话数 |
| `filesTouched` | 观察到的被修改文件路径（不完整，仅 Edit/Write tool 调用，Bash 写入不计） |
| `turns` | 子 Agent 的 assistant 文本轮次，最多保留 `MAX_TURNS = 30` 条 |
| `priorMtime` | 用于 kill 时回滚 lock |

**进度监听**：`makeDreamProgressWatcher` 订阅子 Agent 每条 `assistant` 消息，提取文本块、统计 tool_use 数量、收集 Edit/Write 的 `file_path`，调用 `addDreamTurn` 更新状态。

**kill 路径**：用户在 Shift+↓ 对话框按 `x` → `DreamTask.kill()` → abort 子 Agent → `rollbackConsolidationLock(priorMtime)` 回滚时间戳。

### DreamDetailDialog — 进度对话框

**文件**：`src/components/tasks/DreamDetailDialog.tsx`

- 标题："Memory consolidation"，副标题显示耗时、正在审阅的会话数、已触碰文件数
- 最多展示最近 `VISIBLE_TURNS = 6` 条有文本的轮次，更早的折叠为 "(N earlier turns)"
- 快捷键：`Esc/Enter/Space` 关闭，`x` 停止，`←` 返回列表

### consolidationPrompt.ts — 4 阶段 prompt

**文件**：`src/services/autoDream/consolidationPrompt.ts`

子 Agent 收到的 prompt 分 4 个阶段：

1. **Phase 1 — Orient**：`ls` 记忆目录，读 `MEMORY.md` 索引，略读现有 topic 文件，避免重复创建
2. **Phase 2 — Gather recent signal**：按优先级查找新信息（daily logs > 已漂移的记忆 > transcript grep）
3. **Phase 3 — Consolidate**：合并新信号到现有 topic 文件，绝对日期替换相对日期，删除矛盾事实
4. **Phase 4 — Prune and index**：更新 `MEMORY.md` 索引，保持在 25KB 以内，每条 ≤150 字符，删除过时指针

## 边界条件 & 注意事项

**turns 限制为最近 30 条的原因**：子 Agent 可能运行较长时间产生大量轮次，全量保存会使 React 状态膨胀并拖慢 UI 重渲染。`MAX_TURNS = 30` 是展示用的滑动窗口，不影响子 Agent 实际执行。

**filesTouched 的不完整性**：只捕获 Edit/Write tool_use 中的 `file_path`，Bash 重定向写入的文件不在其中。注释明确标注 "at least these were touched"，不能理解为"只改了这些"。

**与手动 `/dream` 的关系**：手动 `/dream` 在主会话中运行（有完整 tool 权限），AutoDream fork 的子 Agent 只有只读 Bash 权限。两者共用同一份 consolidationPrompt，但 AutoDream 版本在 `extra` 字段追加了 Tool 限制说明和待审阅的 sessionId 列表。

**KAIROS 模式**：KAIROS 模式使用 disk-skill dream，`isGateOpen()` 中直接返回 false，AutoDream 不介入。

**并发安全**：同一机器多个 Claude Code 进程通过 lock 文件的 PID + CAS 写入保证互斥；不同机器（如 remote mode）通过 `getIsRemoteMode()` 直接跳过。
