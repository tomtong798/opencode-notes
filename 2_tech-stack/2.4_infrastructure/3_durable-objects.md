# Durable Objects

## 是什么

Cloudflare 的**强一致有状态 serverless 原语**：每个 Durable Object（DO）全局唯一 ID、单线程串行访问、自带存储——天然适合做"需要串行化的协同点"（同步枢纽、排行榜、presence）。

## 业界类比

- "serverless 界的 Actor 模型"：一个对象 = 一个串行化的状态容器
- Redis 单分区的"每个键一个专属小 Redis"版

## 在 OpenCode 中的角色

- **会话分享（session sharing）**：`packages/function` 的 `SyncServer` DO
  - `POST /share_create`：创建分享（带 secret）
  - `POST /share_sync`：发布会话数据
  - `GET /share_poll`：**WebSocket** 订阅更新（SSE 之外的少数 WS 使用点）
  - `GET /share_data`：取分享数据
  - `POST /share_delete`：删除
- 分享 = 把本地 session 同步到云端给他人查看（只读协作），DO 保证每个分享的写入串行一致

## 与其他技术的关系

- **Cloudflare Workers**：DO 是 Workers 平台的状态原语
- **WebSocket**：`share_poll` 用 WS 推送更新（对比本地 SSE 单向下行为主）
- **SST**：资源编排

## 相关源码

- `packages/function/src/api.ts`（`SyncServer` Durable Object 定义）
