# WebSocket

## 是什么

基于 HTTP 升级握手的**双向全双工**通信协议（RFC 6455）：一条 TCP 连接上双方随时互发消息帧（文本/二进制），带 ping/pong 心跳与关闭握手。`ws://` 明文、`wss://` 加密。

## 业界类比

- SSE 的"双向版"：SSE 只能服务端→客户端且纯 HTTP；WebSocket 升级协议后脱离 HTTP 语义
- "打完电话就不挂的电话线"——对比 HTTP 每次说话都要重新拨号（请求/响应）

## 在 OpenCode 中的角色

仓库原则是**能 SSE 就 SSE**（简单、过代理友好），WebSocket 只用在**真正双向**的场景：

### 产品路径

| 场景 | 位置 | 说明 |
|---|---|---|
| **PTY 终端流** | server `handlers/pty.ts`（HTTP `ctx.request.upgrade` 升级）+ `handlers/pty-socket.ts`（`runPtySocket`：单写者队列保序，replay/实时输出/关闭帧同队列） | TUI/Web/桌面里的**交互式终端**：客户端按键上行、终端输出下行——典型双向流，SSE 干不了（PTY 机制详见 [pty.md](9_pty.md)） |
| 客户端消费 | `packages/client/src/solid/pty.ts`（`new WebSocket(url)`） | 正因需要原生 WS 而非 fetch，`pty.connect`/`persistentPty.connect` 被排除在生成客户端之外（protocol 的 `promiseOmitEndpoints`/`effectOmitEndpoints`） |
| **LLM provider 传输** | core `effect/websocket-constructor.ts`：构造 `Socket.WebSocketConstructor` Layer，带代理支持（`WS_PROXY`/`HTTP_PROXY`/`NO_PROXY` 环境变量解析、本地地址直连） | 部分 provider（如 OpenAI Realtime 类）用 WS 传输；Effect `Socket` 模块统一封装 |
| **云侧会话分享** | `packages/function` 的 `SyncServer` Durable Object：`GET /share_poll` 以 WebSocket 订阅更新 | 分享数据的实时推送 |

### 开发/测试路径

- `packages/simulation`：模拟框架的 control server（`ws` 包的 `WebSocketServer`）、frontend/backend 协议
- Vite HMR：开发态浏览器与 dev server 的热更新通道（`vite-hmr` 子协议）
- e2e mock server、provider 测试磁带（http-recorder 同样录 WS 流量）

## 与其他技术的关系

- **SSE**：本仓库的默认推送通道（状态事件流），WebSocket 仅用于双向场景——两者互补而非竞争，见 [sse.md](7_sse.md)
- **Effect Socket**：core 里 WS 的抽象层（`effect/socket`），配合 Layer 注入构造器
- **Durable Objects**：云侧 WS 的宿主（`share_poll`）
- **OpenTUI / xterm 类终端**：PTY 流的最终渲染端

## 相关源码

- `packages/server/src/handlers/pty.ts`、`pty-socket.ts`、`persistent-pty.ts`
- `packages/client/src/solid/pty.ts`
- `packages/core/src/effect/websocket-constructor.ts`、`effect/app-node-platform.ts`
- `packages/function/src/api.ts`（SyncServer）、`packages/simulation/src/control-server.ts`
