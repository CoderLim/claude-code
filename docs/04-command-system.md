# 命令系统（Command System）

## 概述

命令系统管理所有斜杠命令（`/clear`、`/help`、`/diff` 等），是用户与 Claude Code Harness 直接交互的接口，独立于 LLM 的 tool_use 机制。命令由用户在 REPL 中手动输入触发（部分也可被 LLM 调用），执行结果可以是文本输出、JSX 界面或注入到对话上下文的 prompt。

## 核心职责

- 维护全局命令注册表（`COMMANDS` 数组，memoized，启动时延迟加载）
- 定义三种命令类型：`prompt`（LLM 驱动）、`local`（本地执行）、`local-jsx`（React UI）
- 按 feature flag 和用户类型动态组装可用命令集（与工具系统类似的条件注册模式）
- 合并 Skill、Plugin、MCP 注入的动态命令（`getCommands` 统一出口）
- 提供命令可用性控制（`availability`、`isEnabled`、`isHidden`）

## 架构概览

```
用户输入 /xxx [args]
  ↓
REPL 命令解析（匹配 name 或 aliases）
  ↓
isEnabled() + availability 检查
  ↓
  ├── type: 'prompt'    → getPromptForCommand(args) → 注入 LLM 上下文 → 发 API 请求
  ├── type: 'local'     → load() → call(args, context) → LocalCommandResult
  └── type: 'local-jsx' → load() → call(onDone, context, args) → React.ReactNode（渲染 UI）

命令注册来源：
  COMMANDS()            ← 内置命令（100+ 条）
  getSkillDirCommands() ← ~/.claude/commands/ 和项目 .claude/commands/
  getBundledSkills()    ← 打包内置 Skills
  getPluginSkills()     ← 插件注入 Skills
  getBuiltinPluginSkillCommands() ← 内置插件 Skills
  MCP server 注册的命令
```

## 核心实现

### Command 类型结构

**文件**：`src/types/command.ts`

所有命令共享 `CommandBase`，再结合三种执行模式之一：

```ts
type Command = CommandBase & (PromptCommand | LocalCommand | LocalJSXCommand)

type CommandBase = {
  name: string
  aliases?: string[]
  description: string
  isEnabled?: () => boolean   // 默认 true，feature flag / env 检查
  isHidden?: boolean          // 隐藏于自动补全和 /help
  availability?: ('claude-ai' | 'console')[]  // auth 类型过滤
  kind?: 'workflow'           // workflow 类命令，补全时有徽章
  immediate?: boolean         // 立即执行，不等主对话停止（如 /btw）
  isSensitive?: boolean       // 参数从历史中脱敏
  loadedFrom?: 'commands_DEPRECATED' | 'skills' | 'plugin' | 'managed' | 'bundled' | 'mcp'
  source: SettingSource | 'builtin' | 'mcp' | 'plugin' | 'bundled'
  context?: 'inline' | 'fork' // Skill 执行模式：内联展开 vs 子 Agent 隔离
}
```

**三种命令类型**：

| 类型 | 执行方式 | 典型用途 |
|---|---|---|
| `prompt` | `getPromptForCommand(args, ctx)` 返回 prompt content，注入 LLM 上下文后发 API 请求 | Skills（`/commit`、`/review`、自定义 skill） |
| `local` | `load()` 懒加载模块 → `call(args, ctx)` 返回 `LocalCommandResult` | 轻量本地操作（`/clear`、`/copy`） |
| `local-jsx` | `load()` 懒加载模块 → `call(onDone, ctx, args)` 返回 `React.ReactNode` | 需要 TUI 界面的命令（`/diff`、`/config`、`/mcp`） |

`LocalCommandResult`：`{ type: 'text'; value }` | `{ type: 'compact'; compactionResult }` | `{ type: 'skip' }`

`LocalJSXCommandOnDone`：命令完成回调，支持 `display`（skip/system/user）、`shouldQuery`（完成后是否触发 LLM）、`metaMessages`（插入 meta 消息）

### 命令注册表

**文件**：`src/commands.ts`

`COMMANDS` 通过 `memoize()` 包装（首次调用时构建，后续复用），收集约 100+ 条命令：

**无条件注册的核心命令**（约 60 条）：
`addDir`、`agents`、`branch`、`btw`、`clear`、`color`、`compact`、`config`、`context`、`cost`、`diff`、`doctor`、`effort`、`exit`、`fast`、`files`、`heapDump`、`help`、`ide`、`init`、`keybindings`、`mcp`、`memory`、`model`、`outputStyle`、`plan`、`plugin`、`pr_comments`、`releaseNotes`、`reloadPlugins`、`rename`、`resume`、`review`、`rewind`、`securityReview`、`session`、`skills`、`stats`、`status`、`tasks`、`theme`、`thinkback`、`usage`、`usageReport`、`vim` 等。

**feature flag 条件注册**：

| Feature Flag | 命令 |
|---|---|
| `PROACTIVE` / `KAIROS` | `proactive`、`briefCommand`、`assistantCommand`、`subscribePr`、`sendUserFile` |
| `BRIDGE_MODE` | `bridge`、`remoteControlServerCommand` |
| `VOICE_MODE` | `voiceCommand` |
| `HISTORY_SNIP` | `forceSnip` |
| `WORKFLOW_SCRIPTS` | `workflowsCmd` |
| `CCR_REMOTE_SETUP` | `webCmd` |
| `FORK_SUBAGENT` | `forkCmd` |
| `BUDDY` | `buddy` |
| `ULTRAPLAN` | `ultraplan` |
| `UDS_INBOX` | `peersCmd` |
| `USER_TYPE === 'ant'` | `INTERNAL_ONLY_COMMANDS`（backfillSessions、breakCache、bughunter、commit、ctx_viz 等） |

**动态命令来源**（`getCommands(cwd)` 合并）：
1. `getSkillDirCommands(cwd)` — 扫描 `~/.claude/commands/` 和项目 `.claude/commands/` 目录的 Markdown skill 文件
2. `getBundledSkills()` — 打包进二进制的内置 skills（同步，启动时注册）
3. `getPluginSkills()` — 安装的插件注入的 skill 命令
4. `getBuiltinPluginSkillCommands()` — 启用的内置插件提供的 skill 命令
5. MCP server 注册的工具也可作为命令（`isMcp: true`）

### 命令分类

**编辑与代码操作**：`/diff`、`/rewind`、`/compact`、`/branch`、`/files`、`/copy`

**查看与分析**：`/cost`、`/context`、`/stats`、`/usage`、`/usageReport`（即 `/insights`）、`/doctor`、`/status`

**Agent 与多智能体**：`/agents`、`/session`、`/tasks`、`/btw`、`/fork`、`/peers`、`/teleport`

**配置与设置**：`/config`、`/model`、`/theme`、`/permissions`、`/hooks`、`/mcp`、`/keybindings`、`/vim`、`/outputStyle`、`/privacy-settings`、`/effort`

**工具扩展**：`/skills`、`/plugin`、`/reload-plugins`

**开发辅助**：`/init`、`/plan`、`/review`、`/security-review`、`/commit`、`/commit-push-pr`

**系统**：`/clear`、`/help`、`/exit`、`/login`、`/logout`、`/upgrade`、`/version`

### 命令 vs 工具的核心区别

| 维度 | 命令（Command） | 工具（Tool） |
|---|---|---|
| 触发者 | 用户手动输入 `/xxx` | LLM 在 tool_use block 中选择 |
| 执行层 | Harness 直接执行 | Harness 代理执行，LLM 决策 |
| 输出流向 | 文本/JSX/注入 prompt | tool_result block → LLM |
| 权限检查 | availability / isEnabled | canUseTool（useCanUseTool hook）|
| 交互性 | 可渲染复杂 TUI（local-jsx） | 进度流（onProgress） |

### Skill 命令注册方式

**文件**：`src/skills/loadSkillsDir.ts`（见 10-skills-plugins.md 详述）

Skill 文件（`.md`）通过 frontmatter 解析生成 `PromptCommand`：
- `type: 'prompt'`，`source: 'skills'`（或 `'bundled'`/`'plugin'`）
- `getPromptForCommand` 读取 Markdown 正文，替换 `$ARGUMENTS` 占位符
- `context: 'fork'` 时，skill 在独立子 Agent 中执行（有独立 token 预算）
- `disableModelInvocation: true` 时，LLM 无法主动调用该命令（只有用户可用）
- `paths` 字段支持路径触发：只有模型操作过匹配文件后，该 skill 才出现

`COMMANDS` 数组中的命令名称不会与 skill 重名（`builtInCommandNames` 用于检测冲突，skill 加载时跳过同名内置命令）。

## 关键设计决策

1. **`memoize()` 延迟构建，而非模块加载时初始化**：命令列表依赖 `config`（如 `isUsing3PServices()`），必须在运行时读取，无法在模块初始化时构建，memoize 保证只构建一次。

2. **懒加载（`load()` getter）**：`local` 和 `local-jsx` 类命令通过 `load()` 返回 Promise，延迟 require 重依赖模块（如 `/insights` 的 113KB 图表渲染）。用户不调用就不加载，显著降低启动时间。

3. **`prompt` 命令与 Skill 统一结构**：内置命令（`/commit`、`/review`）和用户 Skill 文件都使用相同的 `PromptCommand` 接口，统一进入 LLM 对话流，无需区分处理。

4. **`immediate: true` 跳过队列**：`/btw` 等设为 immediate 的命令无需等待主对话停止，打字时即可触发，通过 `runForkedAgent` 并行执行，不污染主 session。

## 与其他系统的交互

- **REPL（UI 层）**：解析用户输入，匹配命令名，调用对应 `call()` 或 `getPromptForCommand()`
- **QueryEngine**：`prompt` 类命令的 content 注入 `messages[]`，进入正常 LLM 请求流程
- **Skills 系统**：`getSkillDirCommands` 从磁盘加载 skill 文件，转化为 `PromptCommand` 注入注册表
- **Plugin 系统**：`getPluginSkills` 从插件 manifest 加载 skill 命令，`getBuiltinPluginSkillCommands` 加载内置插件 skill
- **AppState**：`local-jsx` 命令通过 `LocalJSXCommandContext` 访问 `setMessages`、`onChangeAPIKey` 等 UI 回调
