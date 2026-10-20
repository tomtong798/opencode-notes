# 契约先行（Contract-First）+ 代码生成

## 是什么

**先定义接口契约，再生成代码**的开发模式：契约（schema/API 定义）是单一真相源，服务端实现与客户端代码都从它派生；契约变更 → 重新生成，漂移在编译期暴露。

## 业界类比

- OpenAPI/gRPC proto 的"schema 先行"，但契约是 TS 代码而非 YAML
- tRPC 的端到端类型安全，但要发布独立 npm 包的场景（生成物可脱离源仓消费）

## 在 OpenCode 中的角色

**生成流水线**：

```
Effect Schema（packages/schema，领域形状 + durable 事件）
        ↓
HttpApi 定义（packages/protocol，~30 Group，端点/错误全 schema 化）
        ↓ bun run generate（packages/client/script/build.ts）
@opencode/httpapi-codegen
        ↓
生成客户端：promise/（fetch + AsyncIterable SSE）+ effect/（Effect HttpClient）
        ↓ 之上再加包装
shared-events.ts（SSE 去重共享）→ solid/data.ts（store 投影）
```

- **铁律**（根 AGENTS.md）：改 Protocol/Server HttpApi 后必须 `bun run generate`；**禁止手编生成文件**
- 领域契约同样先行：`packages/schema` 是 79 个文件的形状单一真相源（事件清单 `ServerDefinitions` 对外/`Definitions` 对内）
- SDK 则是"组合式先行"：把生成 client + core + server 组合发布

## 与其他技术的关系

- **Effect Schema / HttpApi / httpapi-codegen**：流水线的三块基石（各有专篇）
- **严格分层**：契约先行是分层铁律的执行手段（见 [layered-monorepo.md](4_layered-monorepo.md)）
- **Prettier**：生成物格式化

## 相关源码

- `packages/schema/src/`、`packages/protocol/src/`、`packages/httpapi-codegen/src/`、`packages/client/script/build.ts`、`packages/client/src/{promise,effect,solid}/`
