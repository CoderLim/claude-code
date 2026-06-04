# UI 渲染系统（UI Rendering System）

## 概述

Claude Code 的 UI 层是一套完全自研的终端渲染引擎，以 `src/ink/` 为核心，实现了 React → 虚拟 DOM → yoga 布局 → ANSI 终端输出的完整渲染管线。它并非直接使用 npm 上的 Ink 库，而是在其基础上做了大量定制：双缓冲帧差异渲染、硬件滚动优化（DECSTBM）、字符池（CharPool）、多终端键盘协议支持（Kitty/CSI-u/xterm）、鼠标事件处理、文本选择等。

## 核心职责

- 将 React 组件树通过 React Reconciler 映射到自定义 DOM（`DOMElement`/`TextNode`）
- 使用 Yoga（Facebook's flexbox 布局引擎，native-ts 编译）计算节点尺寸和位置
- 通过 `render-node-to-output.ts` 将 yoga 布局树渲染为 `Output`（ANSI 字符网格）
- 双缓冲（front/back frame）差异渲染：只输出变化的 ANSI 差异序列（`writeDiffToTerminal`）
- `parse-keypress.ts`：支持 VT100/xterm/Kitty/CSI-u 多协议键盘事件解析
- `termio/`：原始终端 I/O 原语（ANSI 序列生成、滚动控制、光标定位等）
- 管理 144+ React 组件（消息列表、权限弹窗、进度条、diff 查看器等）

## 架构概览

```
React 组件树（JSX）
  ↓ react-reconciler（createReconciler）
自定义 DOM（DOMElement / TextNode）
  ↓ yoga-layout（flexbox 计算 → 每个节点的 x/y/width/height）
render-node-to-output.ts（深度优先遍历节点树）
  ↓ 文字切割、颜色化（applyTextStyles）、边框渲染（render-border.ts）
Output（二维字符网格 / 带样式 ANSI 行缓冲）
  ↓ screen.ts（CharPool 字符池 + 双缓冲 Screen）
renderer.ts（front/back frame 差异计算）
  ↓ writeDiffToTerminal（按行对比，发 ANSI 差异序列）
终端（stdout）
```

**帧循环**（`ink.tsx`）：`FRAME_INTERVAL_MS` 节流的 throttle，状态变化触发 React 重渲染，renderer 计算差异后调 `writeDiffToTerminal` 输出到 stdout。

## 核心实现

### 自定义 Ink 框架

**文件**：`src/ink/ink.tsx`、`src/ink/reconciler.ts`

使用 `createReconciler`（`react-reconciler` 包）创建自定义 React 宿主环境，宿主是自己的 DOM：
- `createNode(tagName)` / `createTextNode(text)` 创建虚拟 DOM 节点
- `appendChildNode` / `removeChildNode` / `insertBeforeNode` 修改 DOM 树
- `setAttribute` / `setStyle` / `setTextStyles` 修改节点属性和样式
- `markDirty` 标记 yoga 节点需要重新布局

### render-node-to-output.ts — 节点渲染核心

**文件**：`src/ink/render-node-to-output.ts`

**布局优化**：
- `layoutShifted` 标志：如果本帧任何节点的 yoga position/size 与缓存不同，标记为"布局移位"，触发全损伤重绘；稳态帧（spinner tick、时钟 tick）不移位 → O(变化 cells) 差异
- `ScrollHint`：`ScrollBox` 的 scrollTop 变化时，不重写整个视口，而是发 DECSTBM + SU/SD 硬件滚动序列
- `nodeCache`：缓存上一帧的节点渲染结果，未变化节点直接 blit（`O(1)`），避免重新渲染

**渲染流程**：
1. 遍历 yoga 布局树，获取每个节点的矩形
2. `squashTextNodesToSegments`：将连续文本节点合并为 `StyledSegment[]`
3. `wrapText`：按 terminal 宽度折行（`widestLine`、`stringWidth`）
4. `applyTextStyles`（`colorize.ts`）：注入 ANSI SGR 颜色/样式转义序列
5. `renderBorder`：渲染 box-model 边框（unicode 线框字符）
6. 写入 `Output` 对象

### screen.ts — 双缓冲字符网格

**文件**：`src/ink/screen.ts`

```ts
class CharPool {
  intern(char: string): number  // ASCII 快路径：O(1) 数组查找；非 ASCII 用 Map
  get(index: number): string
}
```

`Screen` 是一个二维字符网格（`CharPool` 引用 + 颜色/超链接元数据），采用**字符串池**（CharPool）技术：
- 相同字符全局共享同一 ID，跨帧 blit 直接比较整数 ID（不做字符串比较）
- ASCII 快路径：`Int32Array[charCode] → index`，避免 Map 查找
- 双缓冲：`frontFrame`（屏幕当前状态）和 `backFrame`（新一帧）交换

`StylePool`：样式对象同理池化，避免 GC 压力。

**差异计算**：`renderer.ts` 对 front/back frame 逐行比较，`screen.diffEach` 找到变化单元格集合，`writeDiffToTerminal` 只发变化单元格的 ANSI 序列。

**屏幕尺寸管理**：通过 `process.stdout.columns`/`rows` + SIGWINCH 信号响应终端尺寸变化，触发全量重渲染。

### termio/ — 终端 I/O 原语

**文件**：`src/ink/termio/`

| 模块 | 内容 |
|---|---|
| `ansi.ts` | 基础 ANSI 常量（ESC、BEL、SEP） |
| `csi.ts` | CSI 序列（光标移动、DECSTBM 滚动区域、Kitty 键盘协议开关） |
| `dec.ts` | DEC 私有序列（alt screen 进入/退出、鼠标跟踪开关、光标隐藏） |
| `esc.ts` | ESC 序列原语 |
| `osc.ts` | OSC 序列（iTerm2 进度条、tab 状态、剪贴板写入、超链接） |
| `sgr.ts` | SGR 颜色/样式序列 |
| `tokenize.ts` | 从终端输入字节流中识别 ANSI 序列边界（Tokenizer） |
| `parser.ts` | 解析 Tokenizer 产出的 token 为高层事件 |

### parse-keypress.ts — 键盘事件解析

**文件**：`src/ink/parse-keypress.ts`

从 `termio/tokenize.ts` 的字节流输出中识别并解析多种终端键盘协议：

| 协议 | 检测正则 | 备注 |
|---|---|---|
| VT100 / ANSI 功能键 | `FN_KEY_RE` | 标准 ESC[ 序列 |
| CSI u（Kitty protocol）| `CSI_U_RE` | `ESC[codepoint;modifierу` |
| modifyOtherKeys（xterm）| `MODIFY_OTHER_KEYS_RE` | `ESC[27;modifier;keycode~` |
| DECRPM | `DECRPM_RE` | 终端模式查询响应 |
| DA1/DA2 设备属性 | `DA1_RE`/`DA2_RE` | 用于检测终端能力 |
| 鼠标事件 | CSI M/m 系列 | 鼠标点击/移动/滚轮 |
| Bracketed paste | `PASTE_START`/`PASTE_END` | 粘贴区域标记 |
| Meta/Alt 键 | `META_KEY_CODE_RE` | `ESC + 字符` |

解析结果为 `ParsedKey`（含 `key`、`ctrl`、`meta`、`shift`、`fn` 等字段）。

### 组件分类（144 个）

**文件**：`src/components/`

| 类别 | 代表组件 |
|---|---|
| **消息渲染** | `MessageList.tsx`、`VirtualMessageList.tsx`（虚拟滚动）、`AssistantMessage.tsx`、`UserMessage.tsx` |
| **工具 UI** | `ToolUseLoader.tsx`、各 Tool 目录下的 `UI.tsx` |
| **权限/对话框** | `permissions/`、`BypassPermissionsModeDialog.tsx`、`TrustDialog/`、`AutoModeOptInDialog.tsx` |
| **进度与任务** | `BashModeProgress.tsx`、`TaskListV2.tsx`、`AgentProgressLine.tsx` |
| **diff 查看** | `StructuredDiff.tsx`、`StructuredDiffList.tsx` |
| **输入** | `TextInput.tsx`、`VimTextInput.tsx`、`BaseTextInput.tsx` |
| **状态栏** | `StatusLine.tsx`、`StatusNotices.tsx` |
| **团队/Agent** | `teams/`、`agents/`、`CoordinatorAgentStatus.tsx`、`TeammateViewHeader.tsx` |
| **导航/设置** | `ThemePicker.tsx`、`TagTabs.tsx`、`wizard/`、`design-system/` |
| **通知** | `ClaudeCodeHint/`、`AutoUpdater.tsx`、`TokenWarning.tsx` |

**VirtualMessageList**：消息列表可能有成百上千条，使用虚拟滚动（只渲染视口内的消息）避免 DOM 节点数量爆炸。

### 键盘管理

**文件**：`src/keybindings/`

从 `~/.claude/keybindings.json` 加载用户自定义快捷键，通过 `useKeybinding` hook 注册。支持 chord 绑定（按键序列）和修饰键组合。

`ink.tsx` 在初始化时根据终端能力决定是否启用：
- Kitty 键盘协议（`ENABLE_KITTY_KEYBOARD`）— 现代终端
- modifyOtherKeys（`ENABLE_MODIFY_OTHER_KEYS`）— xterm/tmux
- 鼠标跟踪（`ENABLE_MOUSE_TRACKING`）— 支持点击和滚轮

## 关键设计决策

1. **完全替代标准 Ink 库而非分叉**：Claude Code 对 Ink 的改动远超 upstream（双缓冲、CharPool、硬件滚动、鼠标支持），维护 fork 难度高，直接在 `src/ink/` 内置一套更实际。

2. **CharPool 字符串池 + 整数 ID 比较**：终端每帧可能有数千个字符单元格，字符串比较开销高。CharPool 将字符变为整数 ID，blit 和 diff 都是整数操作，在高刷新率场景（spinner、流式输出）收益显著。

3. **双缓冲 + 布局变化检测分离差异范围**：`layoutShifted` 标志区分"布局移位帧"和"内容更新帧"。布局不变时，只 diff 文本变化的那几行，发最少的 ANSI 序列；移位时才用全损伤重绘。

4. **VirtualMessageList 按需渲染**：长会话的 messages[] 可能有数百条，全量渲染会让 yoga 计算爆炸。虚拟滚动只计算视口行，保证长对话下的 UI 响应速度。

## 与其他系统的交互

- **AppState / Store**：`useAppState(selector)` 驱动所有 React 组件重渲染；`useSyncExternalStore` 保证并发安全
- **QueryEngine**：流式输出通过 React state 更新触发帧更新，spinner/进度实时渲染
- **Hook 系统**：`useGlobalKeybindings` 注册快捷键；`useIDEIntegration` 注入 MCP 配置
- **Task 系统**：`TaskListV2` 订阅 `AppState.tasks` 显示后台任务进度
- **keybindings.json**：用户自定义快捷键由 `src/keybindings/` 加载并注册到 termio 键盘事件流
