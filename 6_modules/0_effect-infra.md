# Effect 基建详解（LayerNode 依赖图 + global/location 双作用域 + 替换式组装）

> 一句话：**opencode 在 Effect Layer 之上自建的一层"可命名、可依赖、可替换、可分作用域"的依赖图基建（LayerNode），加上 keyed-mutex / WebSocket 构造器 / 平台节点等 Effect 工具箱，让 core 里几十个服务能按 global（进程级）与 location（目录级）两种生命周期组装、共享与热替换。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19）。
> 权威出处：无对应 spec（specs/v2/ 只覆盖 session/tools 等；本模块的"权威"就是 `packages/util/src/effect/layer-node.ts` 源码与类型级错误提示）。
> [返回 README](README.md) | 前置：无（本卷地基） | 关联：[../2_tech-stack/2.2_server/3_effect.md](../2_tech-stack/2.2_server/3_effect.md)（Effect 技术本体）

---

## 0. 模块地图（文件清单 + 行数 + 各节归属）

V2.0.30 的 Effect 基建**横跨两个包**：语言级抽象上移到了 `packages/util`，core 侧只留应用级组装与工具。

| 文件 | 行数 | 内容 | 本文节 |
|---|---|---|---|
| `packages/util/src/effect/layer-node.ts` | 330 | LayerNode 依赖图核心（Tag/Graph/Node/compile） | §2 |
| `packages/util/src/effect/app-node.ts` | 14 | global/location 双 tag 体系 | §3 |
| `packages/util/src/effect/app-node-platform.ts` | 13 | 平台基础节点（FileSystem/Path/HttpClient） | §4.2 |
| `packages/util/src/effect/runtime.ts` | 21 | makeRuntime：Layer → 可运行时序 | §4.3 |
| `packages/util/src/effect/service-use.ts` | 43 | serviceUse：Proxy 惰性服务访问器 | §4.4 |
| `packages/util/src/effect/memo-map.ts` | 3 | 进程级共享 MemoMap | §4.5 |
| `packages/core/src/effect/app-node-builder.ts` | 29 | 应用组装入口：Instance 节点 + build() | §5 |
| `packages/core/src/effect/app-node-platform.ts` | 35 | RequestExecutor/LLMClient/WS 节点 | §4.2 |
| `packages/core/src/effect/keyed-mutex.ts` | 45 | 按 key 的内存互斥锁 | §6 |
| `packages/core/src/effect/websocket-constructor.ts` | 81 | 带代理支持的 WS 构造器 Layer | §7 |

交叉验证文件（不属于本模块但是主要消费方）：

| 文件 | 行数 | 用途 |
|---|---|---|
| `packages/core/src/instance/service.ts` | 19 | Instance.Service + unbound node |
| `packages/core/src/instance.ts` | 181 | ~50 个 location 节点的 graph + `layer()` 编译 |
| `packages/core/src/location-service-map.ts` | 46 | unbound 的 LocationServiceMap 节点 |
| `packages/core/src/location-services.ts` | 87 | `buildLocationServiceMap`（惰性编译宿主） |
| `packages/server/src/routes.ts` | — | `AppNodeBuilder.build` 顶层调用点 |

## 1. 心智模型

五个核心概念：

1. **LayerNode.Graph** —— 给匿名 Effect Layer 加上"名字 + 依赖 + 类型级校验"的图节点（layer-node.ts:49-67）。Effect 原生 Layer 无法表达"这个 Layer 依赖谁、可被谁替换"，LayerNode 补的就是这块。
2. **Tag（global/location）** —— brand 过的作用域标签（layer-node.ts:10-11）。`compile` 时同一 tag 的子图共享一份实例：global = 进程单例（Database、Config.Global），location = 每目录一份（Agent、Tool）。
3. **Replacement** —— `node.replace(x)` 声明"编译时用 x 顶替我，包括所有依赖我的边"（layer-node.ts:81-91）。这是整个系统唯一的覆盖机制：测试替身、vanilla 模式、宿主定制、Location 绑定全走它。
4. **unbound node** —— 只有服务签名没有实现的占位节点（layer-node.ts:214-216）；编译时若无人 replace 它则直接抛错（layer-node.ts:271）。
5. **Instance/LocationServiceMap** —— 用 `LayerMap` 把"每个 location 编译一份 location 图"变成可借还的服务（location-services.ts:20-57）；惰性编译时序详见 [../5_timeline/3_location-lazy.md](../5_timeline/3_location-lazy.md)，本文只讲它挂在哪条基建上。

端到端流转（一个服务从定义到被消费）：

```
[定义] filesystem.ts:219
  makeLocationNode({ service, layer, deps: [FSUtil, Location, ...] })
      │  tags.make("location") = 打上 location 作用域 tag
      ▼
[汇图] instance.ts:59-114
  nodes = [Location.node, Config.node, FileSystem.node, ...约 50 个]
  graph = LayerNode.group(nodes)            ← 有序、只暴露根输出
      │
      ▼
[编译入口] instance.ts:148  layer(ref, options)
  replacements = [vanilla?, 调用方替换, Location.node.replace(boundNode(ref)), ...]
      │
      ▼
[编译] layer-node.ts:238  compile(graph, { replacements, shared: global-tag })
  ├─ resolve: 先应用全部 replace，再 DFS 展开（环检测 :259 / unbound 报错 :271）
  ├─ 同一 implementation Layer 挂不同依赖 → 抛错 :280-287
  └─ shared tag 子图用外层 memoMap；其余每次 build 用新 map :300-311
      │
      ▼
[组装宿主] location-services.ts:12  buildLocationServiceMap
  LayerMap.make((ref) => Instance.layer(ref, { replacements: bindings }))
  └─ 每个 Location.Ref 借出一份编译产物，idleTimeToLive: infinity :56
      │
      ▼
[顶层] app-node-builder.ts:22  build(root, replacements)
  └─ Instance.node.replace(instances) + LocationServiceMap.node.replace(map)
      │
      ▼
[消费] server/routes.ts:151  AppNodeBuilder.build(applicationServices, replacements)
  └─ Session 侧：Instance.Service.provide(session) 把 location 上下文注入 Effect
     （instance/service.ts:12-15）
```

## 2. LayerNode 核心（util/effect/layer-node.ts）

### 2.1 数据模型

```
Definition（判别联合，layer-node.ts:39-47）
  ├─ { kind: "group",    name, dependencies[] }        ← group() 的产物
  ├─ { kind: "unbound",  name }                        ← unbound() 的占位
  └─ { kind: "layer",    name, implementation: Layer, dependencies[] }

GraphValue<A,E,T>     :49   持有 Definition，variance brand 防结构伪造
  └─ NodeValue        :69   多一个 tag；replace() 只在 Node 上
        └─ ProviderNode :94  多一个 mapLayer()（装饰实现、保留接线）
ReplacementValue      :106  { source, target } 对；类型系统保证契约不弱化
```

类型级护栏（编译期就报错，不是运行时）：

| 检查 | 类型 | 错误提示字面量 |
|---|---|---|
| deps 覆盖 layer 所需 R | `CheckDependencies`（:133-137） | `"Missing dependencies"` |
| 替换输出/错误/tag 兼容 | `ReplacementErrors`（:123-131） | `"Missing replacement outputs"` / `"New replacement errors"` / `"Invalid replacement tag"` |
| 替换裸 Layer 必须 closed（R=never） | :129-131 | `"Replacement must be a closed layer"` |
| tag 依赖闭包 | `CheckTags`（:155-159） | `"Invalid tag dependencies"` |

### 2.2 构造 API 逐个

```
make(input)            :188  最常用：{ service, layer, deps, tag? } → Provider
unbound(service, tag)  :214  占位节点；compile 见到未替换的 unbound 直接抛错 :271
group(dependencies)    :219  有序、可结合的组装；只有根输出暴露，依赖保持私有
node.replace(target)   :81   Node→Node（tag 必须相同 :86）或 Node→closed Layer :90
provider.mapLayer(f)   :96   装饰实现但保留依赖接线（仅 layer kind，:98）
tags(config)           :171  一次声明 tag 体系 + 受约束的 make 工厂
```

### 2.3 compile：替换、共享与缓存

`compile(root, { replacements, shared })`（layer-node.ts:238-319）分两阶段：

```
阶段一 resolve（纯图操作，不获取任何资源；注释 :237）
  ├─ replacement Map：source→target，先替换再展开 :266-268
  ├─ 环检测：栈回溯，报 "Cycle detected in layer graph: A -> B -> A" :259-264
  ├─ shared 判定：inherited || (node.tag === shared) :255
  ├─ 去重：同一 implementation Layer 以不同依赖出现 → 抛错 :280-287
  │    （含义：想复用就给同一个 Layer 对象且同一套依赖）
  └─ 两级缓存 cache.shared / cache.local :246-249

阶段二 build（Layer.fromBuild 回调内，Effect 负责获取与终结）
  ├─ local = shared 未设 ? 外层 memoMap : 新 MemoMap :300
  ├─ 每个 node 包 Layer.suspend + provide([ambient, ...deps]) :308
  ├─ shared 节点用外层 memoMap，local 用新 map :309
  │    （★ 全局单例跨 location 共享、location 图每次 build 独立的关键）
  └─ buildContext 用 BuildResult 服务保住精确输出 Context :321-329
       （防 Effect 往 Context 里追加 memo-map 元数据污染输出）
```

`CompileOptions.shared` 的注释（:228）一语道破设计意图：*"Share subgraphs rooted at this tag; give the remaining wiring a fresh map for each build."*

## 3. app-node：双作用域 tag 体系（util/effect/app-node.ts）

```
tags = LayerNode.tags({ location: ["global"], global: [] })   :3-6
  ├─ global   依赖闭包为空       → 进程单例
  ├─ location 依赖 ["global"]    → 每份图一份，可引用全局
  └─ 类型上 location 节点只能依赖 global/无 tag 节点，反向被 CheckTags 拒绝

makeGlobalNode   = tags.make("global")    :11
makeLocationNode = tags.make("location")  :12
```

这正是旧笔记里 `hoist()` 的替代品：V2.0.30 没有 hoist 函数，全局共享改为 `compile` 的 `shared` 参数（instance.ts:159 传入 `Node.tags.values.global`），由编译器内联完成"抽取共享子图"，不再是一个独立图变换步骤。

## 4. 外围工具（runtime / platform / serviceUse / memoMap）

### 4.1 core 侧平台节点（core/src/effect/app-node-platform.ts）

三个 global 节点把网络栈挂进图：

```
requestExecutor  :21-25  RequestExecutor.layer + withoutBunTimeout
  └─ withoutBunTimeout :10-19  关掉 Bun fetch 五分钟静默超时
       （注释 :9——Bun 的超时早于 provider 路由自身的 header/chunk 超时）
llmClient               :27  依赖 requestExecutor
webSocketConstructor    :29-33  挂 §7 的 WS 构造器
```

util 侧 `app-node-platform.ts:9-11` 另提供 FileSystem/Path/HttpClient 三个节点；注释（:1-2）解释为何用深导入——platform-node barrel 会拉进 undici/ioredis/node:sqlite，workerd 装不下。

### 4.2 makeRuntime（util/effect/runtime.ts:5-20）

把 `(service, layer)` 变成带 `runSync/runPromise/runFork/...` 的宿主运行器；运行时惰性创建（`rt ??=`），自动并入 `Observability.layer()` 与全局 memoMap（memo-map.ts:2）。bridge 层（RPC handler、CLI 入口）靠它从 Effect 世界回到 callback 世界。

### 4.3 serviceUse（util/effect/service-use.ts:17-43）

Proxy 实现的惰性服务访问器：`svc.method(args)` 展开成 `tag.use(s => s.method(args))`，只暴露返回 Effect 的方法（类型侧 ServiceUse mapped type，:5-15）。注释（:19-20）明说这是唯一动态边界——TS 知道访问器形状，Proxy 的 key 是运行时值。方法不存在时 `Effect.die`（:32）。

### 4.4 memoMap（util/effect/memo-map.ts:2）

进程级 `Layer.makeMemoMapUnsafe()`，配合 makeRuntime 让多次 run 复用已构建服务。

## 5. core 组装：app-node-builder + Instance

### 5.1 Instance 节点（core/src/effect/app-node-builder.ts:8-20）

```
instances = makeGlobalNode({
  service: Instance.Service,
  layer: Layer.effect(Instance.Service, ... locations.get(session.location) ...),
  deps: [LocationServiceMap.node],
})
```

Instance.Interface（instance/service.ts:11-15）只有一个方法 `provide(session)`：把 session 所属 location 的整份服务上下文 `Effect.provide` 进任意 Effect——这是 location 作用域通往消费方的唯一桥。`node` 本身是 unbound 的 global 节点（instance/service.ts:19），由 build 时替换。

### 5.2 build()（app-node-builder.ts:22-27）

```
build(root, replacements)
  ├─ bindings = [Instance.node.replace(instances), ...replacements]
  └─ LayerNode.compile(root, {
       replacements: [
         LocationServiceMap.node.replace(buildLocationServiceMap(bindings)),  ★ 递归注入
         ...bindings,
       ],
     })
```

★ 处是精髓：LocationServiceMap 拿到的 bindings 里**包含 Instance 替换自身**——每个 location 内部的 `Instance.Service` 由 location-services.ts:75-81 再替换为"从 map 自取"的闭包实现，形成自举。server/routes.ts:142-151 是现实调用样例：standard 替换 + 可选 simulation 替换汇成一个 build。

LocationServiceMap 自身（location-service-map.ts:9-18）：`Context.Service` 包 `LayerMap.LayerMap<Location.Ref, Instance.Services, Instance.Error>`，node 为 unbound global（:18）；`reload`（:20-36）先 invalidate 全部再并发重boot、聚合失败；`canonical`（:39-44）在 Windows 上归一化路径，避免等价 placement 变成不同缓存 key。惰性编译/借还时序细节属于 [../5_timeline/3_location-lazy.md](../5_timeline/3_location-lazy.md)。

### 5.3 Instance.layer（instance.ts:148-181）

单 location 编译的收口处，三件事：

```
layer(ref, options)
  ├─ replacements 有序叠加 :152-157
  │    vanilla 默认（discovery:false，:142-145）< 调用方 < Location/Plugins 绑定
  ├─ compile(graph, { replacements, shared: global-tag }) :159
  ├─ catchCause：DirectoryNotFoundError → 类型化失败，其余 defect 化 :161-171
  └─ tap：成功后注册 LocationLifecycle.shutdown 终结器 + boot 日志 :172-179
```

注释 :147 的不变量：**"One instance is one compiled, fresh copy of the graph standing on a directory."**

## 6. KeyedMutex（core/src/effect/keyed-mutex.ts）

按 key 的内存互斥：同 key 排队、异 key 并行（注释 :14-15）。`users` 计数同时数持有者与等待者，归零即删表项（:30-36，`Effect.ensuring` 保证异常路径也减）。`makeUnsafe` 是纯同步构造，`make` 包成 Effect（:45）。仓内消费方：`bus.ts`、`git.ts`、`session/inbox.ts`、`file-mutation.ts`、`workspace.ts`、`mcp/index.ts` 等——典型如 per-session、per-path 的串行化。

## 7. WebSocketConstructor（core/src/effect/websocket-constructor.ts）

`Layer.succeed(Socket.WebSocketConstructor, fn)`（:58-79），fn 内：

```
(url, input)
  ├─ Bun 在场 → 原生 WebSocket，支持 handshake headers + proxy 选项 :62-70
  │    （注释 :61：信任留在 runtime store，NODE_EXTRA_CA_CERTS 保持叠加语义）
  └─ 否则 NodeWS.WebSocket
       ├─ proxyAgent：wss/https 代理用 HttpsProxyAgent，否则 HttpProxyAgent :52-56
       └─ followRedirects: false :76
            （注释：在 header 跨 origin 前拒绝重定向，调用方安全回落 HTTP）
```

代理选择链（`proxy`，:36-47）：localhost/127.0.0.1/::1 直连 → `NO_PROXY` 匹配（`bypassesProxy`，:22-34，支持 `*`、`host:port`、`.domain` 后缀）→ `WSS_PROXY/WS_PROXY` → `HTTPS_PROXY/HTTP_PROXY` → `ALL_PROXY`。导出的 `{ layer, proxy }`（:81）让代理逻辑可独立复用（如 SSE）。

## 8. 不变量与设计取舍

从代码注释与根 AGENTS.md 风格条款提取：

1. **替换先于获取**（layer-node.ts:237 注释）：resolve 阶段完成全部覆盖判定后才进入构建，"Effect owns acquisition and finalization"——图操作不碰资源。
2. **同一 Layer 对象 = 同一单例**：同 implementation 不同依赖直接抛错（:280-287），不做静默 best-effort 合并。
3. **裸 Layer 替换必须 closed**（R=never，:129-131）：宿主替进来的 Layer 不能自带未满足依赖，把 R 检查前移到 replace 调用点。
4. **global 显式共享、local 默认新鲜**：`shared` tag 决定用哪个 memoMap（:300-309），与 instance.ts:159 配合实现"跨 location 单例、每 location 独立图"，且 lazy LayerMap 继承外层共享缓存（:307 注释）。
5. **类型即文档**：错误字面量（"Missing dependencies" 等）出现在类型位置，接线错误在编译期以可读消息暴露——符合 AGENTS.md "validate once at the boundary" 的精神。
6. **深导入避开重 barrel**：util 与 core 的 platform 文件都以注释解释为何不用 `@effect/platform-node` 全量 barrel（workerd 兼容、Bun 超时）——环境差异在基建层就地消化，不泄漏给业务节点。
7. **缓存键归一化**：LocationServiceMap.canonical（:39-44）在进 RcMap 之前归一化等价路径——"Normalize equivalent placements before they become resource-cache keys"。

## 9. 失败与恢复

本模块有三条独立失败面：

```
编译期（同步、确定性）
  ├─ 环           → throw "Cycle detected ..."            layer-node.ts:261
  ├─ 未绑定 unbound → throw "Unbound layer node: ..."      layer-node.ts:271
  └─ 同 Layer 异依赖 → throw "... use a distinct implementation Layer" :285

构建期（Effect 化）
  ├─ instance.ts:161-171：DirectoryNotFoundError 降为类型化错误，其余 defect
  └─ location-services.ts:28-50：boot 失败的 onExit 路径
       ├─ 失败即逐出缓存项（"Evict once per failed build, before its result
       │   reaches borrowers" :42-43），下次借用触发重建
       ├─ 清理 fork 进 owner scope（:47-48），无借用者的 invalidate 会关闭条目
       │   scope 并 join 本次 lookup fiber，让 owner 收尾（:45-46 注释）
       └─ reload（location-service-map.ts:27-34）：并发重 boot、全结算后再报首个失败

KeyedMutex
  └─ 无失败面：Effect.ensuring 保证 users 计数在任何中断下正确回退（:32-37）
```

## 演进注记（V1 对照，仅模块级差异）

对照 2026-08-03 基线（gitee `5_modules/4.18_effect-infra`）：

- **LayerNode 迁包**：`core/src/effect/layer-node.ts`（333 行）→ `packages/util/src/effect/layer-node.ts`（330 行）。核心符号 `make/unbound/group/replace/compile/tags` 全部存活，签名基本稳定。
- **hoist() 消失**：旧笔记的"阶段 ④ 提升 global（LayerNode.hoist）"不再是独立函数；等价能力并入 `compile` 的 `CompileOptions.shared`（layer-node.ts:228-229）。读旧笔记时需把 hoist 环节映射到 shared 参数。
- **app-node 迁包**：`core/src/effect/app-node.ts` → `util/src/effect/app-node.ts`，`tags/makeGlobalNode/makeLocationNode` 符号未漂移。
- **State.create 存活但不在本模块**：仍在 `core/src/state.ts`（128 → 252 行，大幅增长），消费方为 model.ts:95、worktree/strategies.ts:48、instruction-discovery.ts:60；它已长成独立的注册/重放子系统，不再适合塞进"effect 基建"一篇。
- **InstanceState 退役**：`packages/opencode/src/effect/instance-state.ts` 及整个 `packages/opencode/src/effect/` 目录已删除；按目录缓存职责由 `LocationServiceMap`（LayerMap + RcMap，location-services.ts）承接。
- **run-service 迁移**：旧 `packages/opencode/src/effect/run-service.ts` → `util/src/effect/runtime.ts`（makeRuntime 符号未变），另新增 service-use.ts / memo-map.ts 两个小件。
- **新增件**：keyed-mutex.ts、websocket-constructor.ts、core 侧 app-node-platform.ts 为旧笔记所无；`ProviderNode.mapLayer`、`LayerNode.tags` 的 tag 依赖闭包校验也是新面。

## 相关篇

- [../2_tech-stack/2.2_server/3_effect.md](../2_tech-stack/2.2_server/3_effect.md) —— Effect 语言/库本体（Layer/Context.Service 语义），本文只写 opencode 的用法基建
- [../5_timeline/3_location-lazy.md](../5_timeline/3_location-lazy.md) —— LocationServiceMap 的惰性编译时序（本文 §5.2 的展开）
- [4_session.md](4_session.md) —— Instance.provide 的最大消费方：session 执行链如何借到 location 上下文
- [../5_timeline/3_location-lazy.md](../5_timeline/3_location-lazy.md) —— LocationServiceMap 的惰性编译时序（本文 §5.2 的展开，模块视角见未来 15_location 篇）
