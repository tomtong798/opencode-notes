# SSE（Server-Sent Events）

## 是什么

基于 HTTP 的**服务端单向推送**标准：客户端发起一条普通 GET 长连接，服务端持续写 `data: {...}\n\n` 文本帧。浏览器有原生 `EventSource`，本仓库客户端自行解析。

## 业界类比

- WebSocket 的"只要下行"简化版：单向、纯 HTTP、天然过代理/CDN、自带重连语义
- gRPC streaming / WebSocket 适合双向场景；纯推送场景 SSE 更简单

## 在 OpenCode 中的角色

- **默认的实时事件通道**：`GET /api/event`（protocol `groups/event.ts`）——所有状态同步走 SSE，WebSocket 只留给真正双向的场景（PTY 终端流等，见 [websocket.md](8_websocket.md)）
  - 连接建立先发 `server.connected` 握手事件，15s 心跳（`: heartbeat\n\n`）
  - 每订阅者 4096 事件队列，**慢消费者溢出即断开**（`SubscriberOverflowError`）——背压策略是"断开重连重同步"而非缓冲到底
- **客户端侧**：
  - `SharedEvents`：多订阅者共享一条 SSE 连接（去重）
  - `@opencode/2.1_client/solid` 的 `createData()` 把事件投影为响应式 store（批量 10ms、空闲超时 45s、断线重连后失效缓存重同步）
- 事件过滤：只暴露 `isOpenCodeEvent()` 认可的公共事件（manifest 驱动，见 `specs/v2/event-stream-architecture.md`）
- 注意：durable 事件回放（`sessions.log`）与实例级 live 事件是**两套流**：live-only 增量（文本/推理/工具输入 delta）不保证重放

## 与其他技术的关系

- **HttpApi**：事件端点用 `StreamSse` 编码（见 [effect-httpapi.md](3.2_effect-httpapi.md)）
- **Bus（core）**：服务端 EventFeed 订阅 core Bus → 编码为 SSE 帧
- **SolidJS**：事件最终投影为 store 驱动 UI（见 [solidjs.md](../2.1_client/2_solidjs.md)）
- **WebSocket**：仅用于真正双向的场景——PTY 终端流（客户端按键上行）、部分 LLM provider 传输、云侧会话分享（详见 [websocket.md](8_websocket.md)）；SSE 管状态事件推送，二者互补

## 相关源码

- `packages/server/src/event-feed.ts`、`handlers/event.ts`
- `packages/client/src/shared-events.ts`、`solid/connection.ts`、`solid/data.ts`
- 规范：`specs/v2/event-stream-architecture.md`
