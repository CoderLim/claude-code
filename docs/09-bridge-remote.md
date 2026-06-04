# Bridge 与 Agent 间通信（Bridge & Inter-Agent Communication）

## 概述

Claude Code 支持多种 Agent 间通信机制，从同进程内存队列到跨机器 Anthropic 中继服务，适应不同的 Agent 部署场景。核心是 `SendMessageTool` 的四路路由，将消息分派到正确的底层传输通道。此外，`bridge/` 目录实现了 Remote Control（`/remote-control`）功能，让远端设备（浏览器、手机）可以向本地 Claude Code 发送 prompt。

## 核心职责

- 为同进程 teammate 提供零序列化的内存消息队列（`pendingUserMessages[]`）
- 为跨进程 teammate 提供基于文件锁的 Mailbox（`~/.claude/teams/{team}/inboxes/`）
- 为本机多进程通信提供 Unix Domain Socket（`UDS_INBOX` flag）
- 为跨机器通信提供 Bridge（通过 Anthropic CCR v2 服务中继）
- 实现 `sendMessage "*"` 的 broadcast 机制（发给所有 team 成员）
- 维护 session 注册表，供 `ListPeers` 发现可寻址的 Claude 实例

## 架构概览

```
SendMessageTool.call(input.to, message)
  ↓ parseAddress(input.to)
  ├── scheme: 'other'（裸名称）
  │     ↓ 同进程 teammate（InProcessTeammateTask）
  │         → queuePendingMessage(task.pendingUserMessages[])  [内存]
  │     ↓ 跨进程 teammate（LocalAgentTask）
  │         → writeToMailbox(name, msg, teamName)              [文件 Mailbox]
  │     ↓ "*" broadcast → 遍历 team 成员 → writeToMailbox × N  [文件 Mailbox]
  ├── scheme: 'uds' (UDS_INBOX flag)
  │     → sendToUDSSocket(socketPath, message)                 [Unix Domain Socket]
  └── scheme: 'bridge' (UDS_INBOX flag)
        → getReplBridgeHandle().postInterClaudeMessage()       [CCR v2 中继]
        ※ 永远弹权限确认，bypassPermissions 也无法绕过

Bridge（Remote Control）:
  claude.ai / 移动端 / 远端 Claude Code
    ↓ Anthropic CCR v2 WebSocket/SSE
  replBridge.ts / remoteBridgeCore.ts（REPL 侧）
    → handleIngressMessage → injectUserPrompt → 主 Agent
```

## 核心实现

### 5 种通信方式对比

**文件**：`src/tools/SendMessageTool/SendMessageTool.ts`、`src/utils/teammateMailbox.ts`

| 方式 | 触发条件 | 传输介质 | 跨机器 | 自动允许 |
|---|---|---|---|---|
| 同进程内存队列 | `InProcessTeammateTask` | `AppState.tasks[id].pendingUserMessages[]` | 否 | 是 |
| 文件 Mailbox | 跨进程 teammate 或 `to: name` | `~/.claude/teams/{team}/inboxes/{name}.json` + 文件锁 | 否 | 是 |
| broadcast（`*`）| `to: "*"` | 多个 Mailbox 文件 | 否 | 是 |
| UDS Socket | `to: "uds:/path/to.sock"` | Unix Domain Socket | 否（同机器）| 是 |
| Bridge | `to: "bridge:session_xxx"` | Anthropic CCR v2 SSE/WebSocket | 是 | **否**（永远弹窗）|

### 文件 Mailbox

**文件**：`src/utils/teammateMailbox.ts`

```
路径：~/.claude/teams/{team_name}/inboxes/{agent_name}.json
```

消息结构：
```ts
type TeammateMessage = {
  from: string
  text: string       // 纯文本或 JSON 化的结构化消息
  timestamp: string  // ISO 8601
  read: boolean
  color?: string     // 发送方颜色标识
  summary?: string   // 5-10 字预览（UI 展示用）
}
```

**并发安全**：使用 `lockfile` 模块（retry + exponential backoff）：
- `retries: 10`，`minTimeout: 5ms`，`maxTimeout: 100ms`
- 同一 swarm 下多个 Claude 并发写同一 inbox 时序列化

`useInboxPoller` hook 周期轮询 inbox 文件，新消息作为 `TEAMMATE_MESSAGE_TAG` 注入主 Agent 的消息队列。

### SendMessageTool 四路路由

**文件**：`src/tools/SendMessageTool/SendMessageTool.ts`

`parseAddress(input.to)` 解析 `to` 字段的 scheme：
- `"teammate-name"` → scheme: `'other'`
- `"uds:/path/to/socket"` → scheme: `'uds'`
- `"bridge:session_xxx"` → scheme: `'bridge'`
- `"*"` → 广播

**structured message 协议**（JSON body）：
- `{ type: 'shutdown_request', reason? }` → `handleShutdownRequest`，发送到 mailbox，被接收方解析为优雅终止指令
- `{ type: 'shutdown_response', request_id, approve, reason? }` → `handleShutdownApproval`/`Rejection`
- `{ type: 'plan_approval_response', request_id, approve, feedback? }` → Plan 模式审批

结构化消息**只支持文件 Mailbox**（裸名称路由）；`uds:` 和 `bridge:` 只支持纯文本（跨会话字符串边界）。

### UDS（Unix Domain Socket）

**文件**：`src/setup.ts`（启动时注册）

`UDS_INBOX` feature flag 保护。每个 Claude Code 进程启动时：
1. `m.getDefaultUdsSocketPath()` → `/tmp/cc-socks/{pid}.sock`
2. 注册到 `~/.claude/sessions/{pid}.json`：

```json
{
  "pid": 12345,
  "sessionId": "sess_01...",
  "cwd": "/path/to/project",
  "messagingSocketPath": "/tmp/cc-socks/12345.sock",
  "bridgeSessionId": "bridge_01..."
}
```

**自动允许**：UDS 只在同机器两个独立进程间通信，不经过外部网络，无需用户确认。
**只支持纯文本**：无结构化消息协议。

### ListPeers

（`UDS_INBOX` flag 保护，`src/tools/ListPeersTool/ListPeersTool.ts`）

扫描 `~/.claude/sessions/` 目录，验活（`isProcessRunning(pid)`），返回：
- 同机器其他 Claude Code 实例的 UDS socket 路径
- 远端 Bridge session ID（如果该进程同时连了 Remote Control）

**本地优先**：同一进程如果同时有 UDS 和 bridge，ListPeers 只暴露 UDS 路径（减少 latency，避免通过 Anthropic 服务器绕行）。

### Bridge（Remote Control）

**文件**：`src/bridge/remoteBridgeCore.ts`、`src/bridge/replBridge.ts`

Remote Control 的核心协议（env-less 路径，`tengu_bridge_repl_v2` GrowthBook flag）：

```
1. POST /v1/code/sessions                  （OAuth）→ session.id
2. POST /v1/code/sessions/{id}/bridge      （OAuth）→ { worker_jwt, expires_in, api_base_url, worker_epoch }
   每次 /bridge 调用即为注册（bump epoch），无需单独 /worker/register
3. createV2ReplTransport(worker_jwt, epoch) → SSE 事件流 + CCRClient
4. createTokenRefreshScheduler             → 主动提前刷新 JWT（proactive_refresh）
5. 401 on SSE → rebuildTransport（auth_401_recovery，复用 seq-num）
```

**安全约束（bridge: 路由）**：
- `checkPermissions` 返回 `behavior: 'ask'`（非 allow/deny），且 `decisionReason.type === 'safetyCheck'`，`classifierApprovable: false`
- 这意味着即使 `bypassPermissions` 模式（step 1g）也**无法绕过**此确认弹窗
- 理由：跨机器 prompt 注入必须有用户明确同意

**消息格式**（`BoundedUUIDSet` 防重放）：
- `handleIngressMessage`：从 CCR 收到消息 → 注入主 Agent 用户队列
- `makeResultMessage`：Claude 响应 → 发回 CCR → 原始发送方

### Bridge 状态（AppState）

```ts
AppState.replBridgeEnabled      // 用户期望状态（/config 或 footer 切换）
AppState.replBridgeConnected    // env 注册 + session 创建（"Ready"）
AppState.replBridgeSessionActive // ingress WebSocket 已打开（"Connected"）
AppState.replBridgeReconnecting  // 轮询循环处于 error backoff
AppState.replBridgeConnectUrl   // Ready 状态的连接 URL（?bridge=envId）
AppState.replBridgeSessionUrl   // 用户侧 claude.ai 链接
AppState.replBridgeOutboundOnly // 只转发事件，拒绝入站 prompt（outbound-only 模式）
```

## 关键设计决策

1. **文件锁 Mailbox 而非共享内存或管道**：文件锁（lockfile + retry backoff）实现跨进程序列化，不依赖进程间共享内存或系统级消息队列。好处是任何 Claude 进程崩溃后 inbox 文件仍然完整，幸存者可以读到发送时已完成的消息。

2. **bridge: 路由永远弹权限确认，safetyCheck 绕不过 bypassPermissions**：跨机器发送消息等于向另一台设备注入 prompt，是比本地文件写入更高级别的安全风险。`safetyCheck` 类型的 `decisionReason` 确保绕过权限模式也不能跳过这个确认。

3. **同机器 UDS 优先于 bridge**：`ListPeers` 在同时有 UDS 和 bridge 时只暴露 UDS，因为 UDS 延迟更低（本机 socket），也不经过 Anthropic 服务器（减少审计面）。

4. **env-less bridge 路径（remoteBridgeCore）**：历史上 Remote Control 必须经过 Environments API 层（`initBridgeCore`，~2400 行），存在复杂的 poll/dispatch/heartbeat/register 生命周期。`remoteBridgeCore.ts` 绕过这一层，直接用 OAuth 换取 worker_jwt 并建立 SSE 连接，大幅简化连接流程。

## 与其他系统的交互

- **Task 系统**：InProcessTeammateTask 用内存队列（`queuePendingMessage`）；LocalAgentTask 用 Mailbox（`writeToMailbox`）
- **AppState**：bridge 连接状态存储在 `AppState`（`replBridgeEnabled`/`Connected` 等），UI 组件（`BridgeDialog`）直接订阅
- **Hook 系统**：`useInboxPoller` hook 周期轮询 Mailbox 文件，新消息触发 `AppState.inbox` 更新
- **QueryEngine**：入站消息经 `handleIngressMessage` → 注入 `messages[]`，进入正常 LLM 请求流程
- **权限系统**：bridge 路由的 `checkPermissions` 返回 `safetyCheck` 类型，不可被 bypassPermissions 跳过
