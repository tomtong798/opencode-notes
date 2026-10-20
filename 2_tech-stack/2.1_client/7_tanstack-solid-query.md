# TanStack Solid Query

## 是什么

TanStack Query（原 React Query）的 SolidJS 移植：管理**服务端状态**的获取、缓存、失效、重试、后台刷新的库。把"fetch + loading/error 态 + 缓存一致性"从业务代码里抽走。

## 业界类比

- SWR 的全功能版；"客户端缓存层的事实标准"
- 与 Redux 的分工：Redux 管**客户端**状态，Query 管**服务端**状态

## 在 OpenCode 中的角色

- **Web 端（`@opencode/app`）** 的请求态管理
- TUI 端不用它——由 `@opencode/2.1_client/solid` 的 `createData()`（SSE 事件投影 store）承担等价职责，因为 TUI 的数据源是**事件流**而非请求/响应

## 与其他技术的关系

- **SolidJS**：以 signals/store 实现，`createQuery` 返回响应式资源
- **@opencode/client**：生成的类型化客户端是 Query 的 fetcher 层
- **SSE**：Query 管"拉"，SSE 投影管"推"；Web 端两者并存（拉初始化 + 推增量）

## 相关源码

- `packages/app/src/`（查询封装）
