# httpapi-codegen（@opencode/httpapi-codegen）

## 是什么

opencode 自研的**客户端代码生成器**：读取 Effect HttpApi 定义（编译期反射），生成全类型化客户端代码。私有包，是"契约先行"工作流的执行者。

## 业界类比

- openapi-generator 的"以 HttpApi 树为源"版
- tRPC 的"跨进程仍要发布 npm 包"场景补充：生成物可独立发布

## 在 OpenCode 中的角色

生成流水线（`packages/client/script/build.ts`，即 `bun run generate`）：

```
ClientApi（protocol 定义）
  → compile()                       # 反射编译契约
  → emitPromise()                   # → src/promise/generated/（fetch 风格，SSE 端点返回 AsyncIterable）
  → emitEffectImported()            # → src/effect/generated/（Effect HttpClient 风格）
  → emitEffectShape()               # → src/effect/api/api.ts（类型形状）
```

- **铁律**（根 AGENTS.md）：改了 Protocol 或 Server 的 HttpApi 后必须在 `packages/client` 跑 `bun run generate`；**禁止手编生成文件**
- 生成物差异由 protocol 的 omit 清单控制（如 pty 流式端点从 Promise 客户端剔除）

## 与其他技术的关系

- **Effect HttpApi / Schema**：契约源（见对应篇）
- **@opencode/client**：生成物的宿主包；其上再包 `promise/client.ts`（加 SharedEvents、RPC 适配）与 `solid/data.ts`（store 投影）
- **sdk**：把生成客户端组合 Core+Server 成嵌入式 SDK
- **Prettier**：生成代码经 Prettier 格式化

## 相关源码

- `packages/httpapi-codegen/src/index.ts`（`compile` / `emitPromise` / `emitEffectImported` / `emitEffectShape`）
- `packages/client/script/build.ts`
