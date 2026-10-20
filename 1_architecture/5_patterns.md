# 仓内惯用法与设计模式（Patterns）

> 一句话：**根模式是 Effect Layer/Service 依赖注入（core 里 96 个 Service），GoF 模式在其上自然浮现 11 种；再叠一层仓内独有惯例——同名文件目录、三平台后缀、State transform replay 等。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-16），全部结论带文件路径。
> [返回索引](README.md)

---

## 1. 根模式：Effect Service DI

core/src 里有 **96 个 `Context.Service` 定义**。每个服务的固定骨架（以 `session.ts` 为例）：

```ts
// ① 服务类（Context.Tag）
export class Service extends Context.Service<Service, Interface>()("@opencode/Session") {}

// ② Layer 组装：Effect.gen 内 yield* 获取依赖（绑定具名变量——仓规）
const layer = Layer.effect(Service, Effect.gen(function* () {
  const database = yield* Database.Service   // 具名绑定，禁 yield* (yield* Foo).bar()
  const bus = yield* Bus.Service
  // ...
  return Service.of({ create, prompt, ... })
}))

// ③ 节点接线：声明依赖图
export const node = Node.makeGlobalNode({ service: Service, layer, deps: [Database.node, ...] })
```

**LayerNode 系统**（`packages/util/src/effect/layer-node.ts`）：仓内自研的类型化图组装——`makeGlobalNode`（进程级）vs `makeLocationNode`（Location 作用域），服务以 Node 数组声明依赖，支持替换与解析。这是在 Effect Layer 之上的一层"依赖图元系统"。

## 2. 同名文件 + 目录惯例（24 例）

core/src 顶层有 **24 个** `foo.ts` 与 `foo/` 并存的案例（server 1 例：handlers；tui 0 例）。分工固定：

| 位置 | 放什么 |
|---|---|
| `foo.ts` | ① `export * as Foo from "./foo.js"` 命名空间再导出 ② `Interface`（服务契约）③ `class Service` ④ Layer 组装 ⑤ `node` 接线 |
| `foo/` | 实现细节、子服务、平台适配（`*.bun.ts` 等） |

典型：`session.ts`（Service+Layer，16 个依赖）+ `session/`（42 个实现文件）；`tool.ts` + `tool/`（runtime、mcp、plugin/）；`pty.ts`（接口）+ `pty/`（三平台实现）。**效果：契约与实现物理分离，顶层 .ts 就是每个服务的"目录头文件"**。

### 附：DDD 视角——core 是行为内聚的领域层

用 DDD 透镜看 core，两个可验证的判断：

- **Service 是充血的领域对象，不是贫血 DTO**。以 `session.ts:262-367` 为证：`create/remove/prompt/revert` 都是 `Effect.fn` 领域方法，内含业务规则（准入幂等、级联删除、revert 两阶段语义），而非"数据类 + setter"。领域行为住在对象里，外部只能经 Service 接口调用。
- **core 属"宽松分层"（Relaxed Layering）**：它混入了基础设施（database/pty/filesystem），不是纯领域层——但 Effect Layer 让这些设施全部可替换（测试用 Layer 换实现），领域逻辑因此保持独立。

配套的封装策略：工具类型由 `@opencode/schema/tool`（纯形状定义，102 行）与 `@opencode/plugin` 的注册契约共同表达——**封装边界不在运行时隐藏，而在契约层**：外部拿到的是类型与注册接口，执行细节（解码/编码/bounding）留在 `core/src/tool/runtime.ts` 的注册表内部。

## 3. GoF 清单（以现码验证）

### 确认使用（11 种）

| 模式 | 证据 |
|---|---|
| **Observer** | `bus.ts`（PubSub + typed streams + projector 注册）；SSE 分发 |
| **Factory** | `Session.make()`（session/session.ts:42）；`Tool.make`；`createData()`（client） |
| **Adapter** | `aisdk-native.ts:24-77`（`@ai-sdk/*` → `@opencode/ai/providers/*` 重写映射）；`websocket-constructor.ts`（代理适配）；v1 shim |
| **Strategy** | `worktree/strategies.ts:15-24`（可插拔 worktree 策略）；`model-resolver.ts:137-161`（variant 解析） |
| **Command** | `session/inbox.ts:42-68`（Inbox item：User/Synthetic/Compaction/Move + 生命周期状态） |
| **State Machine** | `2.2_server/service-status.ts`（starting/ready/stopping/failed）；`run-coordinator.ts:50-59`（idle→execution→doorbell）；`compaction.ts` |
| **Memento** | `snapshot.ts:128-213`（git tree 快照 capture/restore）；`session/revert.ts`（stage/clear/commit 两阶段还原） |
| **Registry** | `tool.ts:26-33`（namespace/add/update/remove）；`plugin/hooks.ts:42-59`（按域注册） |
| **Builder** | `session/model-request.ts:62-80`（组装 system/messages/tools/options） |
| **Facade** | 顶层 `session.ts` 门面统一暴露 40+ 子模块（`session.ts:118-237`） |
| **Interpreter** | `codemode/tool.ts:71-80`（受限运行时执行 JS 调工具）——**全仓唯一解释器，专为 CodeMode** |

### 部分吻合（4 种）

| 模式 | 情况 |
|---|---|
| Abstract Factory | `makeGlobalNode`/`makeLocationNode` 是工厂函数但非经典抽象工厂 |
| Mediator | Bus 有点像中介者，但本质是 Observer |
| Chain of Responsibility | `PluginHooks.trigger` 链式回调、`Permission.evaluate` 规则链——形似而非经典责任链 |
| Iterator | 无显式迭代器模式（Effect/Stream 是单子式抽象） |

### 未使用（7+ 种）

**Visitor、Bridge、Decorator、Flyweight、Prototype** 均未找到；Singleton 被 Effect 的"Layer 单例"语义取代（服务天然进程内单例，无需手写）。

**结论**：GoF 确认使用 **11 种**——部分形似（如 Bus 之于 Mediator、hooks 链之于 Chain of Responsibility）不计入，避免"集邮式" inflate 清单。Singleton 已被 Effect 的"Layer 单例"语义天然取代（服务即进程内单例，无需手写）。

## 4. 仓内独有惯例（比 GoF 更重要）

### State transform replay（`core/src/state.ts:160-252`）

可重放的同步状态：transform 注册进 Scope，读取时按序重放全部 transform 重建状态；支持批量（通知合并）与分组（失败组同步摘除）。**这是注册表类服务（Tool/Snapshot/PluginHooks/WorktreeStrategies）的统一底座**——插件注册/注销可完整重演。

### Projector / 事件溯源

`Bus.project(definition, projector)`（bus.ts:882-890）+ `session/projector.ts`：durable 事件在发布事务内折叠进读表。详见 [../2_tech-stack/2.3_data/event-sourcing.md](../2_tech-stack/2.3_data/3_event-sourcing.md)。

### Write-ahead claim（`session/execution.ts:76-86`）

`claimOnCommit`/`releaseOnCommit`：执行开始先落"占用"标记，终端事件释放，崩溃保留供启动恢复。

### 三平台文件后缀（34 例）

`*.bun.ts`（12）/ `*.node.ts`（12）/ `*.workerd.ts`（10）：同一能力按运行时拆文件，由包导出条件（`#runtime-import` 等）选择。完整分布：

| 能力 | 三件套 |
|---|---|
| SQLite | `database/sqlite.{bun,node,workerd}.ts` |
| 文件监听 | `filesystem/fff.{bun,node,workerd}.ts` |
| PTY | `pty/pty.*` + `persistent-pty/binary.*` |
| shell 解析 | `shell/parser-wasm.*` |
| 图片处理 | `image/photon-wasm.*` |
| v1 迁移 | `database/v1-migration.bun.ts` + `v1-migration.noop.ts` |
| 运行时导入 | `util/src/runtime/import.*` |
| TUI 侧 | `attention-sounds.{bun,node}`、`string-width.{bun,node}`、`editor-zed-sqlite.{bun,node}`、`runtime-plugin-support.{bun,node}` |
| 其他 | `util/global-roots.workerd.ts`、`plugin/source.{bun,node}.ts`、`console/resource/resource.node.ts` |

### `.gen.ts` 生成物（3 例）

`core/database/migration.gen.ts`、`schema.gen.ts`、`desktop/src/main/storage/migration.gen.ts`——生成物与手写物同目录、后缀可辨。

### config 模块 self-export（`config/*.ts` 首行）

每个配置模块文件顶部 `export * as ConfigXxx from "./xxx.js"`——AGENTS.md 明文要求的模块自描述模式。

### createSimpleContext（`tui/src/context/helper.tsx:3-26`）

TUI 的 SolidJS context 工厂：`{ context, provider, use }` 三件套，11+ 个 context 模块统一用（2.1_client/args/theme/local/2.3_data/…）。

---

## 5. 设计取舍

- **DI 根模式取代了什么**：手写单例（Singleton）、手动依赖传递、模块级 mock——测试用 Layer 替换服务实现，不用 jest.mock
- **同名文件目录的成本**：import 路径略反直觉（`session.ts` 与 `session/session.ts` 并存），换来的是"读契约只看顶层文件"
- **State replay 为什么不用 Redux 式 store**：注册表需要"插件随时注册/注销且可重演"（Location 换作用域重建全部注册），transform replay 天然支持；普通 UI 状态在客户端另用 SolidJS store——两套状态体系各管一侧
- 红线出处：根 `AGENTS.md`（具名绑定、self-export、单文件单函数偏好）

## 相关篇

- [overview.md](1_overview.md)（各服务在系统中的位置）
- [../2_tech-stack/2.2_server/effect.md](../2_tech-stack/2.2_server/3_effect.md)（Effect 机制详解）
- [../2_tech-stack/2.3_data/event-sourcing.md](../2_tech-stack/2.3_data/3_event-sourcing.md)
- TODO: session-engine.md（write-ahead claim 的执行链上下文）
