# Effect（Effect-TS）

## Effect 家族导航（本篇 + 三个子篇）

本篇讲 Effect-TS **核心机制**（效应/错误通道/Layer/Fiber/Scope），三个子篇讲同族的具体应用，按需取用：

| 篇 | 主题 |
|---|---|
| **3_effect.md**（本篇） | Effect-TS 是什么：四大能力、核心 API 心智模型、在 opencode 中的骨架地位 |
| [3.1_effect-schema.md](3.1_effect-schema.md) | Schema：类型+校验一体，全仓契约的单一真相源 |
| [3.2_effect-httpapi.md](3.2_effect-httpapi.md) | HttpApi：用 Schema 声明式定义 ~30 个 API Group |
| [3.3_effect-cli.md](3.3_effect-cli.md) | effect/cli：`opencode` 命令树 |
| [3.4_effect-guide.md](3.4_effect-guide.md) | 学习指南：教程级讲解，为什么/怎么用/Go 类比（含 4.0.1 频率实测） |
| [3.5_effect-api.md](3.5_effect-api.md) | API 速查：分模块 API 表 + Go 对照 + 自建 LayerNode/LayerMap 专节 |

同族但不属"Effect 用法"的周边：`@effect/sql-*`（数据库桥接，见 [1_sqlite.md](../2.3_data/1_sqlite.md)）、`@effect/opentelemetry`（观测，见 [7_opentelemetry.md](../2.4_infrastructure/7_opentelemetry.md)）。

## 是什么

TypeScript 的**函数式效应系统**框架——opencode 后端最鲜明的技术特征。它把副作用（IO、网络、并发）包成可组合的 `Effect` 值（`Effect<A, E, R>`：成功类型 / 错误类型 / 依赖环境），提供四大能力：

1. **类型化错误**：错误进类型签名，编译期可见、可穷尽处理
2. **依赖注入**：Layer 系统按需组装/组合服务（构造一次、随处 `yield*` 获取），替代手写单例与全局状态
3. **结构化并发**：**Fiber**（轻量绿色线程）支持中断（interruption）、join、并行、超时——中断会沿 fiber 树传播并触发清理
4. **资源安全**：Scope 保证 acquire/release 配对（类比 RAII）

## 业界类比

- 把 Haskell/Scala（ZIO 思想）的效应管理带给 TS
- "加强版异步标准库"：Promise 之上加类型化错误、取消语义、DI 与重试/超时组合子

## 在 OpenCode 中的角色

- **core/server 的骨架**：
  - 工具并行执行、LLM 流式处理 = Fiber（`session/runner/step.ts` 工具调用并行 fiber、结果串行落盘）
  - 权限/配置/存储/Location 等服务 = Layer 注入（`2.2_server/src/routes.ts` 的服务节点树）
  - 可恢复错误（SessionNotFound、Conflict…）= typed error → 协议层 27+ 标签错误
  - Effect interruption 是工具取消的唯一机制（规范：中断不得翻译成模型可见失败，见 `specs/v2/tools.md`）
- 仓库风格约束：Effect generator 内先把服务绑到具名变量、禁嵌套 `yield* (yield* Foo).bar()`；同步逻辑不包 Effect
- lint 层面有专门的 ast-grep 规则族：`lint:effect-patterns`、`lint:effect-simplifications`

## 与其他技术的关系

- **Effect Schema / HttpApi / effect-cli**：同族库，见各篇
- **Bun**：Effect 跑在 Bun 运行时上
- **Drizzle/SQLite**：数据库操作经 `@effect/sql-*` 体系包进 Effect
- **SolidJS**：客户端是 SolidJS（Effect 主要在服务端；client 包另有 Effect 风味客户端）

## 学习提示

读 core 源码的门槛主要在 Effect 语法：`yield*`（执行效应）、`Effect.gen`（do 记法）、`Layer`（服务组装）、`pipe`（组合子流水线）。`.opencode/skills/effect` 技能文件是仓内 Effect 约定的权威说明。

## 相关源码

- `packages/core/src/`（全目录）、`packages/server/src/routes.ts`
- `.opencode/skills/effect`、`script/ast-grep/`（Effect 规则）
