# Skills 与 Plugins 系统（Skills & Plugins System）

## 概述

Skills 系统让用户和组织可以用 Markdown 文件定义自定义斜杠命令（`/commit`、`/review` 等），文件通过 frontmatter 描述元数据，正文成为注入 LLM 的 prompt。Plugins 系统是更重量级的扩展机制，以 JS/TypeScript 模块形式提供 skill + hooks + MCP server 的组合能力，并在 `/plugin` UI 中展示，允许用户启用/禁用。

## 核心职责

- **Skills**：从磁盘目录（`~/.claude/skills/`、`.claude/skills/`）扫描 Markdown 文件，解析 frontmatter，注册为 `PromptCommand`
- **Bundled Skills**：打包进 CLI 二进制的内置 skills（`verify`、`simplify`、`remember`、`loop` 等），通过 `registerBundledSkill` 编程式注册
- **Plugins**：JS 模块扩展，可提供 skills、hooks、MCP servers 的组合；`@builtin` 后缀标识内置插件（用户可开关）
- **搜索路径优先级**：policy > user > project（多层从近到远）；bare 模式只扫 `--add-dir`
- **Workflow（WORKFLOW_SCRIPTS flag）**：比 Skill 更底层的脚本文件，由 harness 直接运行而非 Claude 推理

## 架构概览

```
CLI 启动
  ↓ initBundledSkills()    ← src/skills/bundled/index.ts
  ↓ initBuiltinPlugins()   ← src/plugins/bundled/index.ts

getCommands(cwd) 调用时（REPL 初始化）：
  ↓ getSkillDirCommands(cwd)  ← src/skills/loadSkillsDir.ts，memoize
    ├── policySettings: {managed}/.claude/skills/
    ├── userSettings:   ~/.claude/skills/
    ├── projectSettings: .claude/skills/ (从 cwd 向上到 $HOME)
    ├── --add-dir:      {dir}/.claude/skills/
    └── legacy:         .claude/commands/*.md（loadSkillsFromCommandsDir）
  ↓ getBundledSkills()       ← 运行时注册的 bundledSkills[]
  ↓ getPluginSkills()        ← 外部安装插件提供的 skills
  ↓ getBuiltinPluginSkillCommands() ← 启用的内置插件提供的 skills

Skill 执行流程：
  /skill-name [args]
    → getPromptForCommand(args, ctx)
    → 解析 $ARGUMENTS 占位符替换
    → 执行 !`...` shell 命令注入（非 MCP skill）
    → 返回 ContentBlockParam[] → 注入 LLM 上下文 → 发 API 请求
```

## 核心实现

### Skill 文件格式与 frontmatter 字段

**文件**：`src/skills/loadSkillsDir.ts`

技能文件放在 `{dir}/skill-name/SKILL.md`（新格式）或 `{dir}/commands/skill-name.md`（遗留格式）。

**frontmatter 字段（完整）**：

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | string | 显示名称（不影响命令名） |
| `description` | string | 一行描述，用于自动补全和 ToolSearch |
| `when_to_use` | string | 详细使用场景说明，LLM 决定是否调用时参考 |
| `allowed-tools` | string[] | 限制 skill 执行时可用的工具 |
| `argument-hint` | string | 自动补全时显示的参数提示（灰色） |
| `arguments` | string/string[] | 命名参数，支持 `$ARGUMENTS` 或 `$ARG1`/`$ARG2` |
| `model` | string | 指定运行时使用的模型（`'inherit'` = 继承主循环模型） |
| `disable-model-invocation` | bool | true 时 LLM 不可主动调用（只有用户可输入 `/skill-name`） |
| `user-invocable` | bool | false 时用户也不可直接输入（仅 LLM 调用） |
| `context` | `'inline'` / `'fork'` | `'fork'` = 在独立子 Agent 中执行（独立 token 预算） |
| `agent` | string | fork 模式下使用的 agent 类型 |
| `effort` | string/int | 努力程度（影响 thinking 预算）：`low/normal/high/max` |
| `version` | string | Skill 版本号（显示在 `/skills` 列表） |
| `hooks` | HooksSettings | Skill 调用时注册的临时 hooks |
| `paths` | string[] | 路径触发：只有操作过匹配文件后，该 skill 才可见 |
| `shell` | FrontmatterShell | 内联 shell 命令（`!` 命令）的执行配置 |

**`$ARGUMENTS` 占位符**：正文中的 `$ARGUMENTS` 在运行时替换为用户参数；`$CLAUDE_SKILL_DIR` 替换为 skill 目录路径；`$CLAUDE_SESSION_ID` 替换为当前 session ID。

**内联 shell 命令**（`!`\`...\`）：正文中的 `` !`cmd` `` 在 skill 加载时执行，结果内联到 prompt（MCP skill 禁用此功能，安全隔离）。

### 搜索路径优先级

**文件**：`src/skills/loadSkillsDir.ts` → `getSkillDirCommands(cwd)`

优先级从高到低（高优先级同名 skill 覆盖低优先级）：
1. `{managed}/.claude/skills/`（`policySettings`，`CLAUDE_CODE_DISABLE_POLICY_SKILLS` 可禁用）
2. `~/.claude/skills/`（`userSettings`）
3. `.claude/skills/`（`projectSettings`，从 cwd 向 $HOME 查找所有父目录）
4. `{--add-dir}/.claude/skills/`（命令行额外目录）
5. `.claude/commands/*.md`（遗留格式，`loadSkillsFromCommandsDir`）

**bare 模式（`--bare`）**：跳过自动发现，只扫 `--add-dir` 路径（若 `projectSettingsEnabled` 且有 `additionalDirs`）。

**去重**：使用 `realpath` 解析 symlink，避免同一文件通过不同路径被重复注册（`getFileIdentity`）。

**加锁**（`isRestrictedToPluginOnly`）：policy 可强制只允许 plugin skills，拦截所有其他 skill 加载来源。

### Bundled Skills

**文件**：`src/skills/bundled/index.ts`、`src/skills/bundledSkills.ts`

通过 `registerBundledSkill(definition)` 在启动时同步注册，存入进程内 `bundledSkills[]` 数组：

| skill | 功能 |
|---|---|
| `update-config` | 通过自然语言更新 settings.json |
| `keybindings` | 配置快捷键（help skill） |
| `verify` | 验证代码改动是否符合预期 |
| `debug` | 调试辅助 |
| `lorem-ipsum` | 测试文本生成（内部工具） |
| `skillify` | 将对话总结为 skill 文件 |
| `remember` | 手动保存记忆条目 |
| `simplify` | 代码简化审查 |
| `batch` | 批量操作 |
| `stuck` | 当 Claude 卡住时的诊断辅助 |
| `dream`（KAIROS flag）| 记忆整合 |
| `loop`（AGENT_TRIGGERS flag）| 周期性触发（cron-like）|
| `schedule-remote-agents`（AGENT_TRIGGERS_REMOTE）| 调度远端 Agent |
| `claude-in-chrome`（自动检测）| Chrome 浏览器集成 |
| `claudeApi`、`claudeApiContent` | Claude API SDK 辅助 |

`BundledSkillDefinition` 支持 `files` 字段（附带文件，首次调用时 extract 到磁盘 `~/.claude/bundled-skills/{name}/`），skill prompt 前缀注入 "Base directory: {dir}"，让模型可以 Read/Grep 这些文件。

### Plugin 系统

**文件**：`src/plugins/builtinPlugins.ts`、`src/plugins/bundled/index.ts`

Plugin 是比 Skill 更重量级的扩展，一个 Plugin 可以包含：
- 多个 Skill 命令
- Hooks（PreToolUse、PostToolUse 等）
- MCP server 配置

**Plugin ID 格式**：
- 内置插件：`{name}@builtin`
- 市场插件：`{name}@{marketplace}`

**内置插件注册**（`registerBuiltinPlugin`）：
```ts
type BuiltinPluginDefinition = {
  name: string
  defaultEnabled?: boolean   // 默认启用状态
  isAvailable?: () => boolean // 是否在当前环境可用
  skills?: BundledSkillDefinition[]
  hooks?: HooksSettings
  mcpServers?: ...
}
```

启用状态持久化到 `settings.json` 的 `enabledPlugins` 字段：`{ [pluginId]: boolean }`。用户通过 `/plugin` UI 切换。

**外部插件**（`getPluginCommands`）：安装在 `~/.claude/plugins/` 下的 JS 模块，`loadPluginCommands` 加载其导出的 `skills` 和 `commands` 数组。

### Skill vs Plugin vs Workflow 区别

| | Skill | Plugin | Workflow（WORKFLOW_SCRIPTS flag） |
|---|---|---|---|
| 本质 | Markdown 文件 | JS 模块（多组件） | 脚本文件 |
| 执行者 | Claude 读 prompt 后推理 | Harness（hooks/MCP）+ Claude（skills）| Harness 直接运行 |
| 出现在 `/plugin` UI | 否 | 是（可开关） | 否 |
| 自动发现 | 是（扫目录） | 是（扫 `~/.claude/plugins/`）| 是（内置） |
| 用户可创建 | 是 | 否（需打包）| 否 |
| `context: fork` | 支持 | N/A | 总是隔离 |

## 关键设计决策

1. **frontmatter 而非单独 JSON 配置文件**：skill 的 metadata 和 content 在同一文件，便于人工阅读和 LLM 理解；frontmatter 解析复用了 CLAUDE.md 的 `parseFrontmatter` 逻辑，减少重复。

2. **`context: fork` 隔离 skill 的 token 预算**：默认 inline 模式下，skill 的执行 tokens 计入主对话。fork 模式下，skill 在独立子 Agent 执行，有独立 token 预算，适合长时间运行的复杂 skill（如 `/verify`、`/security-review`）。

3. **MCP skill 禁止内联 shell 命令**：MCP skill 来自远端服务器，不可信。`loadedFrom !== 'mcp'` 作为唯一门控，防止远端 skill 利用 `` !`...` `` 注入执行本地命令。

4. **Bundled skills 支持 `files` 字段**：某些 skill 需要附带辅助脚本（`.sh`/`.py`），通过 `files` 字段打包进二进制，首次调用时 extract。这避免了要求用户手动安装脚本，同时保留了 skill 可以依赖外部工具的能力。

## 与其他系统的交互

- **命令系统**：`getSkillDirCommands` / `getBundledSkills` / `getPluginSkills` 被 `getCommands(cwd)` 调用，最终合并进命令注册表
- **QueryEngine**：`PromptCommand.getPromptForCommand(args, ctx)` 返回 prompt content，注入 `messages[]`，触发 LLM 请求
- **Hook 系统**：skill frontmatter 的 `hooks` 字段由 `registerSkillHooks` 在调用时临时注册到 `AppState.sessionHooks`
- **Memory 系统**：`remember` 内置 skill 直接调用 memory 写入接口
- **Task 系统**：`loop` skill（AGENT_TRIGGERS）创建周期性触发任务；Workflow 创建 `LocalWorkflowTask`
