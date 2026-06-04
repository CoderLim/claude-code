# Memory 系统（Memory System）

## 概述

Memory 系统让 Claude Code 在会话之间保留上下文——用户偏好、项目决策、反馈规则等无法从代码库中推导出的信息。每条记忆是一个带有 YAML frontmatter 的 Markdown 文件，存储在磁盘上的固定目录，由 MEMORY.md 索引文件汇总。查询时通过 Sonnet 模型从最多 200 个文件中选出最相关的至多 5 条附加到上下文。

## 核心职责

- 按 4 种类型（user / feedback / project / reference）存储和分类记忆
- 维护 MEMORY.md 索引文件（上限 200 行 / 25 KB），超出则警告
- 通过 `scanMemoryFiles`（最多 200 个文件）+ `findRelevantMemories`（Sonnet 选 ≤5 条）实现相关性召回
- 标记陈旧记忆（>1 天）显示新鲜度警告，防止过期信息被误当事实
- 支持 Team Memory（`TEAMMEM` feature flag），私有记忆与团队共享记忆合并使用

## 架构概览

```
会话结束 / 轮次结束
  ↓ autoDream / extractMemories（background forked agent）
记忆文件 (.md)
  ├── ~/.claude/projects/{sanitized-git-root}/memory/
  │     ├── MEMORY.md               ← 索引，≤200行 / ≤25KB
  │     ├── user-2026-03-01-prefs.md
  │     ├── feedback-no-mock-db.md
  │     └── project-auth-rewrite.md
  └── [Team Memory] ~/.claude/teams/{team}/memory/ (TEAMMEM flag)

查询时（新轮次开始）：
  MEMORY.md → 注入 system prompt（buildMemoryPrompt）
  findRelevantMemories(query, dir)
    → scanMemoryFiles(dir) → 最多 200 个 .md，读 frontmatter，按 mtime 排序
    → Sonnet sideQuery → 选最相关 ≤5 个
    → 以 system-reminder 注入 messages[]（含新鲜度警告）
```

## 核心实现

### 4 种记忆类型

**文件**：`src/memdir/memoryTypes.ts`

| 类型 | scope | 存储时机 | 使用时机 |
|---|---|---|---|
| `user` | 总是私有 | 了解用户角色/偏好/知识时 | 需要根据用户背景定制回答时 |
| `feedback` | 默认私有；项目级规范偏团队 | 用户纠正做法或确认非显然方案时 | 防止重复犯同类错误 |
| `project` | 偏团队 | 了解进行中工作、目标、决策、BUG 时 | 理解请求背景、预判协作问题 |
| `reference` | 通常团队 | 了解外部系统资源位置时（Linear、Grafana 等）| 用户提及外部系统时 |

**不应存储**：可从代码库推导的信息（代码模式、架构、文件路径）、git 历史、调试解决方案、CLAUDE.md 已记录的内容、临时任务状态。即使用户明确要求也遵守此规则——引导用户提取真正"非显然"的部分。

**文件格式**：
```markdown
---
name: 记忆标题
description: 单行描述（用于相关性判断，要具体）
type: user | feedback | project | reference
---

记忆正文
feedback/project 类型建议格式：规则/事实，然后 **Why:** 行 + **How to apply:** 行
```

### 存储路径

**文件**：`src/memdir/paths.ts`

路径解析优先级（从高到低）：
1. `CLAUDE_COWORK_MEMORY_PATH_OVERRIDE` 环境变量（Cowork/SDK 全路径覆盖）
2. `settings.json` 中 `autoMemoryDirectory` 字段（支持 `~/` 展开；**projectSettings 被排除**，防止恶意 repo 重定向到 `~/.ssh`）
3. 默认路径：`{memoryBaseDir}/projects/{sanitized-git-root}/memory/`
   - `memoryBaseDir` = `CLAUDE_CODE_REMOTE_MEMORY_DIR` env var 或 `~/.claude`
   - `sanitized-git-root`：项目 git canonical root 的 sanitize 结果（同 worktree 共享同一目录）

路径安全校验（`validateMemoryPath`）：拒绝相对路径、根/近根路径（length < 3）、UNC 路径（`\\server\share`）、Windows 盘符根（`C:`）、空字节。

Assistant 模式日志路径：`{autoMemPath}/logs/YYYY/MM/YYYY-MM-DD.md`（日记式追加，由 `/dream` skill 周期整合）

### MEMORY.md 索引限制

**文件**：`src/memdir/memdir.ts`

- 最大 **200 行**（`MAX_ENTRYPOINT_LINES`）
- 最大 **25 KB**（`MAX_ENTRYPOINT_BYTES`）
- `truncateEntrypointContent(raw)` 先按行截断，再按字节截断，截断后附警告
- 整个 MEMORY.md 内容注入 system prompt 的 `buildMemoryPrompt` 部分

### scanMemoryFiles — 目录扫描

**文件**：`src/memdir/memoryScan.ts`

```ts
scanMemoryFiles(memoryDir, signal): Promise<MemoryHeader[]>
```

- `readdir(memoryDir, { recursive: true })` 列出所有 `.md` 文件（排除 MEMORY.md）
- 并发读取每个文件的前 30 行（`readFileInRange`）解析 frontmatter
- 按 `mtime` 降序排列，截取最新 **200 个**（`MAX_MEMORY_FILES`）
- 返回 `MemoryHeader[]`（含 `filename`、`filePath`、`mtimeMs`、`description`、`type`）

### findRelevantMemories — 相关性召回

**文件**：`src/memdir/findRelevantMemories.ts`

```ts
findRelevantMemories(query, memoryDir, signal, recentTools?, alreadySurfaced?): Promise<RelevantMemory[]>
```

**两步流程**：
1. `scanMemoryFiles(memoryDir)` → 最多 200 个记忆头部
2. `sideQuery`（Sonnet 小模型）选相关文件：
   - 入参：用户 query + 记忆文件名和 description 清单 + 最近使用的工具列表
   - 返回：至多 **5** 个文件名
   - 逻辑：API 文档类记忆（用户正在使用的工具）跳过；警告/gotcha/已知问题类保留
3. 已在本轮展示过的文件（`alreadySurfaced`）提前过滤，避免占用 5 个名额

返回 `{ path, mtimeMs }[]`，`mtimeMs` 供调用方计算新鲜度警告。

### memoryAge — 新鲜度机制

**文件**：`src/memdir/memoryAge.ts`

- `memoryAgeDays(mtimeMs)` → 距今天数（0=今天，向下取整）
- `memoryFreshnessText(mtimeMs)` → 记忆 **>1 天**时返回警告文本，≤1 天返回空
- 警告内容示例：`"This memory is 47 days ago. Memories are point-in-time observations, not live state — claims about code behavior or file:line citations may be outdated."`
- 以 `<system-reminder>` 标签注入，触发模型对陈旧记忆保持怀疑态度

### isAutoMemoryEnabled — 启用优先级链

**文件**：`src/memdir/paths.ts`

优先级从高到低（第一个有定义的值生效）：
1. `CLAUDE_CODE_DISABLE_AUTO_MEMORY=1` → OFF；`=0` → ON
2. `CLAUDE_CODE_SIMPLE`（--bare 模式）→ OFF
3. CCR 模式（`CLAUDE_CODE_REMOTE=true`）且未设置 `CLAUDE_CODE_REMOTE_MEMORY_DIR` → OFF
4. `settings.json` 中 `autoMemoryEnabled` 字段
5. 默认：ON

### Team Memory

**文件**：`src/memdir/teamMemPaths.ts`（`TEAMMEM` feature flag 保护）

启用后，记忆系统变为私有 + 团队两层目录：
- 私有目录：同默认路径
- 团队目录：`~/.claude/teams/{team}/memory/`（通过 symlink 指向团队共享存储）

`TYPES_SECTION_COMBINED`：包含 `<scope>` 标签的 prompt，引导 Claude 区分私有/团队记忆。
`TYPES_SECTION_INDIVIDUAL`：单目录模式（无 scope 标签）的简化 prompt。

路径安全：`sanitizePathKey` 拒绝 null 字节、URL-encoded traversal、Unicode 规范化攻击（fullwidth `．．／`）、反斜杠、绝对路径。

## 关键设计决策

1. **frontmatter description 驱动相关性**，而非全文向量检索：Sonnet 只看 description 字段决定是否召回，而非全文。这意味着 description 质量决定召回效果；但避免了向量数据库依赖，离线可用。

2. **MEMORY.md 作为索引而非全量存储**：将所有记忆摘要写入 MEMORY.md 注入 system prompt（总量有限），完整记忆文件按需通过 `findRelevantMemories` 加载。两层结构在上下文窗口用量和召回率之间取得平衡。

3. **新鲜度警告 >1 天，而非 >N 天**：记忆描述的是"某个时间点观察到的状态"，代码在持续演化。1 天已经足够让代码变化，且频繁提醒有助于模型建立"记忆可能过期"的默认心智模型。

4. **projectSettings 排除自定义路径**：防止恶意 `.claude/settings.json` 将记忆目录重定向到敏感路径（如 `~/.ssh`）。只有 policySettings/localSettings/userSettings 可以设置 `autoMemoryDirectory`。

## 与其他系统的交互

- **DreamTask（Task 系统）**：autoDream 触发 forked Agent 执行记忆整合，更新 MEMORY.md 和各记忆文件
- **QueryEngine**：每轮开始时 `buildMemoryPrompt` 把 MEMORY.md 注入 system prompt；`findRelevantMemories` 在用户提问后动态附加相关记忆到 messages[]
- **FileReadTool**：读取记忆文件时，`memoryFreshnessNote(mtimeMs)` 在输出前追加新鲜度提醒
- **settings.json**：`autoMemoryEnabled`、`autoMemoryDirectory` 字段控制启用状态和路径
- **Skills / `/memory` 命令**：提供 UI 管理记忆（查看、删除、手动添加）
