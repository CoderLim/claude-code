# Claude Code 架构总览

> 基于 2026 年 3 月 npm sourcemap 泄露事件的源码逆向分析。

---

## 什么是 Claude Code

Claude Code = **Harness** + **LLM**。

LLM（Claude 模型）负责推理与决策——调哪个工具、生成什么回复。Harness 是围绕 LLM 的全部运行时基础设施：组装 messages、执行工具、管理权限、渲染 UI、调度后台任务。两者的接缝在 `query.ts`——整个代码库里唯一真正向 Anthropic API 发请求的地方。`QueryEngine.ts` 在外面跑主循环，把 harness 和 LLM 粘在一起。

---

## 整体架构：三层模型

```
┌──────────────────────────────────────────────┐
│  UI 层       React / Ink                      │
│              REPL、对话框、状态栏、/diff TUI  │
├──────────────────────────────────────────────┤
│  Harness 层  QueryEngine.ts                   │
│              tools / hooks / tasks            │
│              permissions / memory / plugins   │
├──────────────────────────────────────────────┤
│  LLM 层      query.ts → Anthropic API         │
│              Claude 模型（无状态推理）         │
└──────────────────────────────────────────────┘
```

| 层 | 核心文件 | 职责 |
|---|---|---|
| UI 层 | `components/`、`screens/`、`ink.ts` | 渲染对话、工具调用、进度、diff 等 React/Ink 组件 |
| Harness 层 | `QueryEngine.ts`、`tools/`、`tasks/`、`hooks/`、`memdir/` | 主循环、工具执行、后台任务、钩子、权限、记忆 |
| LLM 层 | `query.ts`、`services/api/` | 构造 API 请求、流式解析、重试、compact、cache 控制 |

---

## 启动序列

```
main.tsx
  profileCheckpoint('main_tsx_entry')   // 启动性能打点
  startMdmRawRead()                     // MDM 企业配置并行读（macOS/Windows）
  startKeychainPrefetch()               // keychain OAuth + API key 并行读（~65ms 优化）
  init()                                // 认证、trust dialog、telemetry 初始化
  fetchBootstrapData()                  // 远程配置（模型列表、feature flags 种子）
  initializeGrowthBook()                // GrowthBook feature flags 初始化
  launchRepl()                          // 渲染 React/Ink App + REPL
    → QueryEngine 就绪，等待用户输入
```

`startMdmRawRead` 和 `startKeychainPrefetch` 在模块顶层副作用中提前触发，与后续 ~135ms 的模块加载并行执行，是启动耗时的主要优化点。

---

## 子系统全景图

QueryEngine 是运行时核心，向下协调 9 个子系统：

```
                    ┌─────────────────┐
                    │   QueryEngine   │  主循环、multi-turn 编排
                    └────────┬────────┘
           ┌─────────────────┼─────────────────┐
           │                 │                 │
    ┌──────▼──────┐  ┌───────▼──────┐  ┌──────▼──────┐
    │   query.ts  │  │    tools/    │  │   tasks/    │
    │  LLM API    │  │  工具执行层  │  │  后台任务   │
    └─────────────┘  └──────────────┘  └─────────────┘
           │                 │                 │
    ┌──────▼──────┐  ┌───────▼──────┐  ┌──────▼──────┐
    │  services/  │  │    hooks/    │  │   memdir/   │
    │compact/mcp/ │  │  生命周期钩子│  │  记忆管理   │
    └─────────────┘  └──────────────┘  └─────────────┘
           │                 │                 │
    ┌──────▼──────┐  ┌───────▼──────┐  ┌──────▼──────┐
    │  plugins/   │  │   skills/    │  │   state/    │
    │  MCP 插件   │  │  Skill 指令  │  │  AppState   │
    └─────────────┘  └──────────────┘  └─────────────┘
```

| 子系统 | 目录 | 说明 |
|---|---|---|
| LLM 接口 | `query.ts`、`services/api/` | API 调用、流式解析、重试、compact |
| 工具执行 | `tools/`、`Tool.ts` | Bash、文件读写、MCP、Agent 等 50+ 工具 |
| 后台任务 | `tasks/`、`Task.ts` | Shell/Agent/Workflow/Monitor/Dream 任务 |
| 生命周期钩子 | `hooks/` | PreToolUse、PostToolUse、SessionStart 等事件 |
| 记忆管理 | `memdir/` | MEMORY.md 加载、自动记忆整合（autoDream） |
| MCP 插件 | `plugins/`、`services/mcp/` | 动态加载外部 MCP server 工具 |
| Skill 系统 | `skills/` | Markdown 指令文件，注册为 slash command |
| 应用状态 | `state/AppState.ts` | 全局可变状态（tasks、messages、permissions） |
| Agent 通信 | `tools/AgentTool/`、`remote/` | 文件 Mailbox、UDS、Bridge 三路通信 |

---

## 核心数据流

一次完整对话的端到端流程：

```
1. 用户输入
   └─ REPL（UI 层）捕获，触发 QueryEngine.runQuery()

2. 组装请求
   └─ fetchSystemPromptParts()   系统提示（memory + 项目 CLAUDE.md + 工具描述）
   └─ processUserInput()         用户消息预处理（附件、图片、文件引用）
   └─ addCacheBreakpoints()      标记 prompt cache 断点（每次请求只放一个）

3. 调用 LLM（query.ts）
   └─ 流式接收 AssistantMessage
   └─ 实时渲染到 UI

4. 工具调用循环
   └─ LLM 返回 tool_use block
   └─ canUseTool() 权限检查（hooks / permission mode）
   └─ 执行工具，收集 ToolResult
   └─ 追加到 messages[]，发起下一轮请求
   └─ 直到无 tool_use 或达到 maxTurns

5. 轮次结束
   └─ handleStopHooks()          触发 Stop 钩子
   └─ 保存 CacheSafeParams       供 /btw、speculation 复用
   └─ recordTranscript()         写 session 存储
   └─ flushSessionStorage()      落盘

6. UI 更新
   └─ 渲染最终回复，等待下一次用户输入
```

**副线并行流程**（不阻塞主对话）：
- **Speculation**：用户打字时后台预执行，命中则直接采用结果
- **/btw**：旁路提问，独立 API 调用，不污染主 messages
- **后台任务**：LocalShellTask / DreamTask 等在独立进程/协程中运行

---

## 设计哲学

### LLM 无状态
LLM 本身不持有任何对话状态。每次请求都由 harness 把完整 messages 历史重新组装后发送。状态完全在 harness 侧（AppState、session 存储）管理。

### Prompt Cache 优先
每次请求精确放置一个 `cache_control` 标记，最大化服务端 KV cache 命中。fork agent（/btw、speculation）复用主线程参数（CacheSafeParams），保证字节完全相同，不产生新 cache entry 也不污染主线程缓存前缀。

### 插件化工具
工具通过统一的 `Tool` 接口注册，MCP 工具动态加载后与内置工具无差异对待。Skill 以 Markdown 文件形式存在，注册为 slash command 后由 Claude 读取并推理执行。

### 推测执行（Speculation）
用户打字期间，harness 在后台预执行最可能的下一条 prompt。文件写操作重定向到 overlay 目录（copy-on-write），接受时批量覆盖，拒绝时直接删除，主文件系统零污染。仅对 Anthropic 内部员工开启（`USER_TYPE === 'ant'`）。

---

## 文档导航

| 编号 | 文件 | 主题 |
|---|---|---|
| 01 | `01-query-engine.md` | QueryEngine 主循环与多轮对话编排 |
| 02 | `02-query.md` | query.ts：LLM API 调用、流式解析、compact |
| 03 | `03-tools.md` | 工具系统：注册、执行、权限检查 |
| 04 | `04-tasks.md` | 后台任务系统（Shell/Agent/Workflow/Monitor/Dream） |
| 05 | `05-hooks.md` | Hooks 系统：生命周期钩子与三种执行方式 |
| 06 | `06-agent-communication.md` | Agent 间通信：Mailbox / UDS / Bridge |
| 07 | `07-memory.md` | 记忆系统：memdir、MEMORY.md、autoDream |
| 08 | `08-prompt-cache.md` | Prompt Cache：cache_control、CacheSafeParams、break 检测 |
| 09 | `09-bridge-remote.md` | 桥接与远程：Agent 间通信5种方式、WebSocket、ListPeers |
| 10 | `10-skills-plugins.md` | Skills + Plugins：加载机制、内置技能、插件架构 |
| 11 | `11-prompt-cache.md` | 专题：Prompt Cache 原理与实现（KV Cache、TTL、Cache Break 检测） |
| 12 | `12-speculation.md` | 专题：推测执行原理（Overlay、Boundary、Pipelining） |

> 当前已有详细分析：`internals.md`（Task 系统、Hooks、Agent 通信、/diff、/btw、Speculation、Prompt Cache、Monitor）
