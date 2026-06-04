# Claude Code 文档体系 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将 claude-code 架构分析整理为 13 篇结构化文档，替换现有 internals.md。

**Architecture:** 每篇文档分两层——架构概述（快速理解）+ 源码级深挖（实现细节）。文档全部平铺在 `docs/` 下，以编号文件名区分。子系统文档（01~10）用统一模板；专题文档（11~12）用原理解析结构。

**Tech Stack:** Markdown，源码参考 `src/`，现有分析参考 `docs/internals.md`

---

## 文档模板（每个 Task 均遵循）

### 子系统文档模板（01~10）

```markdown
# [系统名称]

## 概述
[一段话：是什么、解决什么问题、在整体架构中的位置]

## 核心职责
- [职责1]
- [职责2]
- [职责3]

## 架构概览

[文字版组件关系图]

[关键数据流箭头描述]

## 核心实现

### [模块1]
**文件**：`src/path/to/file.ts`
[关键函数/类 + 实现要点]

### [模块2]
...

## 关键设计决策
[值得注意的设计取舍和 Why]

## 与其他系统的交互
- **[系统名]**：[交互方式]
```

### 专题文档模板（11~12）

```markdown
# [专题名]

## 是什么 & 解决什么问题
## 工作原理
## 实现细节
### [模块]
**文件**：`src/path/to/file.ts`
[细节]
## 边界条件 & 注意事项
```

---

## Task 0：初始化目录

**Files:**
- 确认：`docs/` 目录已存在

- [ ] **Step 1：确认目录结构**

```bash
ls ~/AI/daily/claude-code/docs/
```

预期：只有 `internals.md` 和 `superpowers/`，无其他 `.md` 文件。

- [ ] **Step 2：确认源码目录可访问**

```bash
ls ~/AI/daily/claude-code/src/ | wc -l
```

预期：显示若干目录（约 40+）。

---

## Task 1：00-overview.md — 整体架构总览

**Files:**
- Create: `docs/00-overview.md`
- Read: `src/QueryEngine.ts`, `src/query.ts`, `src/main.tsx`
- Reference: `docs/internals.md` 的「整体架构」和「三层模型」部分

- [ ] **Step 1：读取关键源码**

```bash
head -80 ~/AI/daily/claude-code/src/QueryEngine.ts
head -50 ~/AI/daily/claude-code/src/query.ts
head -50 ~/AI/daily/claude-code/src/main.tsx
```

- [ ] **Step 2：创建文档**

创建 `docs/00-overview.md`，内容结构：

```markdown
# Claude Code 架构总览

## 什么是 Claude Code

Claude Code = Harness + LLM。
- **LLM**：Claude 模型本身，负责推理和决策
- **Harness**：除 LLM 之外的所有运行时基础设施

接缝在 `query.ts`——唯一真正发 API 请求的地方。

## 整体架构：三层模型

\```
UI 层      → React/Ink（REPL、组件、对话框）
Harness 层 → QueryEngine、tools、hooks、tasks、permissions
LLM 层     → Anthropic API（query.ts）
\```

## 启动序列

\```
main.tsx
  startMdmRawRead()
  startKeychainPrefetch()
  init()
  fetchBootstrapData()
  initializeGrowthBook()
  launchRepl()
    → QueryEngine 就绪，等待用户输入
\```

## 子系统全景图

\```
QueryEngine（对话引擎）
  ├── Tool System（45+ 工具）
  ├── Command System（100+ 斜杠命令）
  ├── Task System（后台任务调度）
  ├── Hook System（权限 + UI 事件）
  ├── State Management（AppState）
  ├── Memory System（跨会话记忆）
  ├── UI Rendering（Ink 终端框架）
  ├── Bridge & Remote（Agent 间通信）
  └── Skills & Plugins（扩展系统）
\```

## 核心数据流

\```
用户输入
  → Command System 解析斜杠命令 / 直接进入 QueryEngine
  → context.ts 注入 system prompt（git 状态、文件扫描）
  → memdir/ 注入 Memory（MEMORY.md + 相关记忆）
  → query.ts 发送 API 请求
  → LLM 响应 → Tool 调用分派
  → Tool 执行结果写回 messages[]
  → AppState 更新
  → Ink UI 重新渲染
\```

## 设计哲学

- **LLM 无状态**：Harness 维护完整 messages[]，每次请求重新组装
- **Prompt Cache**：服务端缓存 KV 矩阵，前缀相同时复用计算（见 `11-prompt-cache.md`）
- **插件化**：Tool / Command / Task 均为独立注册，feature flag 控制灰度
- **推测执行**：用户打字时后台预执行，提交时直接复用（见 `12-speculation.md`）

## 文档导航

| 文件 | 内容 |
|------|------|
| `01-query-engine.md` | 对话引擎：主循环、消息组装、上下文管理 |
| `02-tool-system.md` | 工具系统：45+ 工具框架与执行流程 |
| `03-task-system.md` | 任务系统：后台任务调度与生命周期 |
| `04-command-system.md` | 命令系统：100+ 斜杠命令注册与分派 |
| `05-hook-system.md` | Hook 系统：权限、IDE 集成、快捷键 |
| `06-state-management.md` | 状态管理：AppState 与响应式更新 |
| `07-memory-system.md` | Memory：跨会话记忆存储与检索 |
| `08-ui-rendering.md` | UI 渲染：自定义 Ink 终端框架 |
| `09-bridge-remote.md` | 桥接与远程：Agent 间通信机制 |
| `10-skills-plugins.md` | Skills + Plugins：扩展系统 |
| `11-prompt-cache.md` | 专题：Prompt Cache 原理与实现 |
| `12-speculation.md` | 专题：推测执行原理与实现 |
```

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/00-overview.md && git commit -m "docs: add 00-overview.md — 整体架构总览"
```

---

## Task 2：01-query-engine.md — 对话引擎

**Files:**
- Create: `docs/01-query-engine.md`
- Read: `src/QueryEngine.ts`, `src/query.ts`, `src/query/tokenBudget.ts`, `src/query/stopHooks.ts`, `src/context.ts`
- Reference: `docs/internals.md` 的「整体架构」部分

- [ ] **Step 1：读取关键源码**

```bash
wc -l ~/AI/daily/claude-code/src/QueryEngine.ts ~/AI/daily/claude-code/src/query.ts
head -100 ~/AI/daily/claude-code/src/QueryEngine.ts
head -100 ~/AI/daily/claude-code/src/query.ts
cat ~/AI/daily/claude-code/src/query/tokenBudget.ts
cat ~/AI/daily/claude-code/src/query/stopHooks.ts
```

- [ ] **Step 2：创建文档**

创建 `docs/01-query-engine.md`，按子系统模板，重点覆盖：
- QueryEngine 主循环（消息组装 → API 请求 → tool 调用分派 → 再循环）
- `query.ts` 是唯一发 API 的接缝
- `context.ts` 如何生成动态 system prompt（git 状态、工具列表、memory）
- Token Budget 管理（`tokenBudget.ts`）
- Stop Hooks（`stopHooks.ts`）
- 上下文压缩（context compaction）触发条件

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/01-query-engine.md && git commit -m "docs: add 01-query-engine.md"
```

---

## Task 3：02-tool-system.md — 工具系统

**Files:**
- Create: `docs/02-tool-system.md`
- Read: `src/Tool.ts`, `src/tools.ts`, `src/tools/BashTool/`, `src/tools/AgentTool/`, `src/tools/FileEditTool/`

- [ ] **Step 1：读取关键源码**

```bash
cat ~/AI/daily/claude-code/src/Tool.ts
head -80 ~/AI/daily/claude-code/src/tools.ts
ls ~/AI/daily/claude-code/src/tools/
```

- [ ] **Step 2：创建文档**

创建 `docs/02-tool-system.md`，重点覆盖：
- `Tool.ts` 基类接口（name、description、inputSchema、call()）
- `tools.ts` 注册表（feature flag 控制工具可用性）
- 45+ 工具按类别分组：文件操作、代码执行、AI 集成、网络、任务系统、模式控制
- 工具调用流程：LLM 输出 tool_use → QueryEngine 分派 → 工具执行 → 结果写回 messages[]
- 权限检查时机（`useCanUseTool`）

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/02-tool-system.md && git commit -m "docs: add 02-tool-system.md"
```

---

## Task 4：03-task-system.md — 任务系统

**Files:**
- Create: `docs/03-task-system.md`
- Read: `src/Task.ts`, `src/tasks.ts`, `src/tasks/LocalShellTask/`, `src/tasks/LocalAgentTask/`, `src/tasks/DreamTask/`
- Reference: `docs/internals.md` 的「Task 系统」部分

- [ ] **Step 1：读取关键源码**

```bash
cat ~/AI/daily/claude-code/src/Task.ts
cat ~/AI/daily/claude-code/src/tasks.ts
ls ~/AI/daily/claude-code/src/tasks/
cat ~/AI/daily/claude-code/src/tasks/types.ts
```

- [ ] **Step 2：创建文档**

创建 `docs/03-task-system.md`，重点覆盖：
- 6 种 Task 类型及各自适用场景
- Task 生命周期（pending → running → completed/failed）
- LocalShellTask：bash 命令后台执行，stdout 写文件 fd，1s 轮询
- LocalAgentTask vs InProcessTeammateTask 的区别
- DreamTask：自动记忆整合（autoDream）
- 任务通知机制（priority: 'later' vs 'next'）

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/03-task-system.md && git commit -m "docs: add 03-task-system.md"
```

---

## Task 5：04-command-system.md — 命令系统

**Files:**
- Create: `docs/04-command-system.md`
- Read: `src/commands.ts`, `src/commands/` 下若干代表性命令目录

- [ ] **Step 1：读取关键源码**

```bash
wc -l ~/AI/daily/claude-code/src/commands.ts
head -60 ~/AI/daily/claude-code/src/commands.ts
ls ~/AI/daily/claude-code/src/commands/ | head -30
```

- [ ] **Step 2：创建文档**

创建 `docs/04-command-system.md`，重点覆盖：
- `commands.ts` 注册表（100+ 命令统一导入）
- 命令结构（name、description、handler）
- 命令分类：编辑类（diff/commit）、查看类（review/usage）、代理类（agents/teammates）、配置类、工具类
- 命令与工具的区别：命令是用户输入的斜杠指令，工具是 LLM 调用的能力
- Skill 命令的注册方式（`/skill-name` 补全时标注 `(skill)`）

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/04-command-system.md && git commit -m "docs: add 04-command-system.md"
```

---

## Task 6：05-hook-system.md — Hook 系统

**Files:**
- Create: `docs/05-hook-system.md`
- Read: `src/hooks/useCanUseTool.tsx`, `src/hooks/useGlobalKeybindings.tsx`, `src/hooks/useIDEIntegration.tsx`
- Reference: `docs/internals.md` 的「Hooks 系统」部分

- [ ] **Step 1：读取关键源码**

```bash
ls ~/AI/daily/claude-code/src/hooks/ | head -30
wc -l ~/AI/daily/claude-code/src/hooks/*.tsx 2>/dev/null | sort -rn | head -10
```

- [ ] **Step 2：创建文档**

创建 `docs/05-hook-system.md`，重点覆盖：
- 两种 Hook 层：React hooks（UI 逻辑）vs settings.json hooks（shell 命令钩子）
- settings.json hooks：支持的 23 个事件（SessionStart/End、PreToolUse/PostToolUse 等）
- 三种 hook 执行方式：execAgentHook、execHttpHook、函数回调
- `useCanUseTool`：工具权限检查流程
- `useGlobalKeybindings`：快捷键注册
- `useIDEIntegration`：IDE 选区同步

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/05-hook-system.md && git commit -m "docs: add 05-hook-system.md"
```

---

## Task 7：06-state-management.md — 状态管理

**Files:**
- Create: `docs/06-state-management.md`
- Read: `src/state/AppState.tsx`, `src/state/AppStateStore.ts`, `src/state/store.ts`, `src/state/selectors.ts`

- [ ] **Step 1：读取关键源码**

```bash
wc -l ~/AI/daily/claude-code/src/state/*.ts ~/AI/daily/claude-code/src/state/*.tsx
head -80 ~/AI/daily/claude-code/src/state/AppState.tsx
cat ~/AI/daily/claude-code/src/state/store.ts
```

- [ ] **Step 2：创建文档**

创建 `docs/06-state-management.md`，重点覆盖：
- AppState 结构（messages、tasks、permissions、currentAgent 等核心字段）
- AppStateStore：状态存储与变更管理
- `onChangeAppState`：响应式监听器
- `selectors.ts`：派生状态计算
- 状态与 UI 渲染的联动（AppState 变更 → Ink 重渲染）
- tasks 中 pendingMessages（in-process teammate 通信的内存队列）

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/06-state-management.md && git commit -m "docs: add 06-state-management.md"
```

---

## Task 8：07-memory-system.md — Memory 系统

**Files:**
- Create: `docs/07-memory-system.md`
- Read: `src/memdir/memoryTypes.ts`, `src/memdir/paths.ts`, `src/memdir/memdir.ts`, `src/memdir/memoryScan.ts`, `src/memdir/findRelevantMemories.ts`, `src/memdir/memoryAge.ts`
- Reference: `docs/internals.md` 的「Memory 系统」整节（已有详细分析）

- [ ] **Step 1：读取关键源码**

```bash
cat ~/AI/daily/claude-code/src/memdir/memoryTypes.ts
cat ~/AI/daily/claude-code/src/memdir/paths.ts
head -60 ~/AI/daily/claude-code/src/memdir/memdir.ts
cat ~/AI/daily/claude-code/src/memdir/memoryAge.ts
```

- [ ] **Step 2：创建文档**

创建 `docs/07-memory-system.md`，重点覆盖（已有详细源码分析，直接迁移整理）：
- 4 种 Memory 类型（user/feedback/project/reference）
- 文件格式（frontmatter + 正文）
- 存储路径：`~/.claude/projects/<git-root>/memory/`
- MEMORY.md 索引（200行/25KB 上限，自动加载）
- `scanMemoryFiles()`：最多 200 个文件，按修改时间排序
- `findRelevantMemories()`：Sonnet 模型驱动，最多召回 5 条
- 新鲜度机制（>1天带陈旧警告）
- `isAutoMemoryEnabled()` 启用优先级链
- Team Memory（feature flag 保护）

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/07-memory-system.md && git commit -m "docs: add 07-memory-system.md"
```

---

## Task 9：08-ui-rendering.md — UI 渲染

**Files:**
- Create: `docs/08-ui-rendering.md`
- Read: `src/ink/` 目录概览, `src/components/` 目录概览

- [ ] **Step 1：读取关键源码**

```bash
ls ~/AI/daily/claude-code/src/ink/
ls ~/AI/daily/claude-code/src/components/ | head -20
wc -l ~/AI/daily/claude-code/src/ink/ink.tsx 2>/dev/null || wc -l ~/AI/daily/claude-code/src/ink/*.tsx 2>/dev/null | sort -rn | head -5
```

- [ ] **Step 2：创建文档**

创建 `docs/08-ui-rendering.md`，重点覆盖：
- 自定义 Ink 框架（基于 React，替代标准 Ink 库）
- 三层渲染：React 组件树 → Ink 虚拟 DOM → 终端 ANSI 输出
- `render-node-to-output.ts`：节点渲染核心
- `termio/`：终端 I/O，处理键盘输入和光标控制
- `parse-keypress.ts`：键盘事件解析
- 146 个组件分类（布局、对话、Agent 状态、差异展示等）
- `screen.ts`：屏幕尺寸管理与自适应

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/08-ui-rendering.md && git commit -m "docs: add 08-ui-rendering.md"
```

---

## Task 10：09-bridge-remote.md — 桥接与远程系统

**Files:**
- Create: `docs/09-bridge-remote.md`
- Read: `src/bridge/replBridge.ts`（头部）, `src/bridge/remoteBridgeCore.ts`, `src/remote/RemoteSessionManager.ts`
- Reference: `docs/internals.md` 的「Agent 间通信」整节

- [ ] **Step 1：读取关键源码**

```bash
ls ~/AI/daily/claude-code/src/bridge/
head -60 ~/AI/daily/claude-code/src/bridge/replBridge.ts
head -60 ~/AI/daily/claude-code/src/bridge/remoteBridgeCore.ts
ls ~/AI/daily/claude-code/src/remote/
```

- [ ] **Step 2：创建文档**

创建 `docs/09-bridge-remote.md`，重点覆盖（直接迁移 internals.md 的「Agent 间通信」）：
- 5 种通信方式对比表（同进程/文件Mailbox/UDS/Bridge/broadcast）
- 文件 Mailbox：`~/.claude/teams/{team}/inboxes/{agent}.json`，文件锁保并发安全
- `SendMessageTool` 四路路由（teammate-name / `*` / uds: / bridge:）
- UDS（Unix Domain Socket）：同机器两进程互发，自动允许
- Bridge（Remote Control）：跨机器经 Anthropic CCR 中转，永远需要用户确认
- `ListPeers`：session 注册表（`~/.claude/sessions/{pid}.json`）
- 结构化协议消息（shutdown_request / plan_approval_response）

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/09-bridge-remote.md && git commit -m "docs: add 09-bridge-remote.md"
```

---

## Task 11：10-skills-plugins.md — Skills + Plugins

**Files:**
- Create: `docs/10-skills-plugins.md`
- Read: `src/skills/loadSkillsDir.ts`（头部）, `src/skills/bundledSkills.ts`, `src/plugins/builtinPlugins.ts`

- [ ] **Step 1：读取关键源码**

```bash
ls ~/AI/daily/claude-code/src/skills/
head -60 ~/AI/daily/claude-code/src/skills/loadSkillsDir.ts
cat ~/AI/daily/claude-code/src/skills/bundledSkills.ts
ls ~/AI/daily/claude-code/src/plugins/
cat ~/AI/daily/claude-code/src/plugins/builtinPlugins.ts
ls ~/AI/daily/claude-code/src/skills/bundled/
```

- [ ] **Step 2：创建文档**

创建 `docs/10-skills-plugins.md`，重点覆盖：
- Skill vs Plugin 的区别（Skill 是 Markdown 指令，Plugin 是 JS 扩展）
- Skill 加载流程：`loadSkillsDir.ts` 扫描目录、读取 frontmatter、注册为斜杠命令
- Skill 搜索路径优先级（用户目录 > 项目目录 > 内置）
- 内置 19 个 Skills（`bundled/` 目录）
- Plugin 架构：`builtinPlugins.ts` 注册，支持自定义工具/命令
- Workflow 与 Skill 的区别：Skill 是 Markdown（Claude 读后执行），Workflow 是脚本（harness 直接运行）

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/10-skills-plugins.md && git commit -m "docs: add 10-skills-plugins.md"
```

---

## Task 12：11-prompt-cache.md — Prompt Cache 专题

**Files:**
- Create: `docs/11-prompt-cache.md`
- Read: `src/utils/` 下 cache 相关文件（`grep -r "promptCache\|cache_control\|cacheBreak" src/ --include="*.ts" -l`）
- Reference: `docs/internals.md` 的「Prompt Cache」整节

- [ ] **Step 1：定位相关源码**

```bash
grep -r "promptCache\|cache_control\|cacheBreak\|addCacheBreakpoints\|skipCacheWrite" ~/AI/daily/claude-code/src/ --include="*.ts" -l 2>/dev/null
grep -r "CacheSafeParams\|promptCacheBreak" ~/AI/daily/claude-code/src/ --include="*.ts" -l 2>/dev/null
```

- [ ] **Step 2：创建文档**

创建 `docs/11-prompt-cache.md`，按专题模板，重点覆盖（直接迁移 internals.md 的「Prompt Cache」）：
- Attention KV 是什么（Transformer Q/K/V 矩阵 + KV Cache 原理）
- Prompt Cache = KV Cache 跨请求持久化
- `cache_control` 标记：每次请求只放 1 个，正常请求放最后一条，skipCacheWrite 放倒数第二条
- TTL 两档：默认 5 分钟，特定用户 1 小时（GrowthBook 控制，session 启动时锁定）
- `addCacheBreakpoints` 逻辑
- `CacheSafeParams`：fork agent（/btw、speculation）复用主线程参数保证命中
- `skipCacheWrite` 语义：只读缓存，不产生新 entry
- Cache Break 检测：两阶段（发请求前哈希 + 收响应后检查 cache_read_input_tokens 降幅）

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/11-prompt-cache.md && git commit -m "docs: add 11-prompt-cache.md"
```

---

## Task 13：12-speculation.md — 推测执行专题

**Files:**
- Create: `docs/12-speculation.md`
- Read: `grep -r "speculation\|Speculation\|overlay" src/ --include="*.ts" -l`
- Reference: `docs/internals.md` 的「Speculation」整节

- [ ] **Step 1：定位相关源码**

```bash
grep -r "speculation\|Speculation\|startSpeculation\|handleSpeculation" ~/AI/daily/claude-code/src/ --include="*.ts" -l 2>/dev/null
grep -r "overlay\|copyOverlay\|safeRemoveOverlay" ~/AI/daily/claude-code/src/ --include="*.ts" -l 2>/dev/null
```

- [ ] **Step 2：创建文档**

创建 `docs/12-speculation.md`，按专题模板，重点覆盖（直接迁移 internals.md 的「Speculation」）：
- 推测执行定义：用户打字时预执行，提交时直接采用结果
- 启用条件：仅 `process.env.USER_TYPE === 'ant'`（Anthropic 内部员工）
- 完整流程：PromptSuggestion → startSpeculation → runForkedAgent → 接受/拒绝
- 文件 Overlay：写操作落在 `~/.claude/tmp/speculation/{pid}/{id}/`，Copy-on-write
- 工具许可表（只读工具允许，写操作看权限模式）
- 4 种 Boundary 类型（complete / bash / edit / denied_tool）
- Pipelining：speculation 完成后立即启动下一轮预执行链

- [ ] **Step 3：Commit**

```bash
cd ~/AI/daily/claude-code && git add docs/12-speculation.md && git commit -m "docs: add 12-speculation.md"
```

---

## Task 14：清理 internals.md

**Files:**
- Delete: `docs/internals.md`

- [ ] **Step 1：确认所有 13 篇文档已创建**

```bash
ls ~/AI/daily/claude-code/docs/*.md | grep -v internals
```

预期：列出 00-overview.md 到 12-speculation.md，共 13 个文件。

- [ ] **Step 2：删除 internals.md**

```bash
cd ~/AI/daily/claude-code && git rm docs/internals.md
```

- [ ] **Step 3：最终 Commit**

```bash
cd ~/AI/daily/claude-code && git commit -m "docs: 完成文档体系重组，废弃 internals.md"
```

- [ ] **Step 4：验证最终目录结构**

```bash
ls ~/AI/daily/claude-code/docs/*.md
```

预期输出：
```
docs/00-overview.md
docs/01-query-engine.md
docs/02-tool-system.md
docs/03-task-system.md
docs/04-command-system.md
docs/05-hook-system.md
docs/06-state-management.md
docs/07-memory-system.md
docs/08-ui-rendering.md
docs/09-bridge-remote.md
docs/10-skills-plugins.md
docs/11-prompt-cache.md
docs/12-speculation.md
```
