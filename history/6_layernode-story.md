# LayerNode 诞生史：一个自建装配图基建的 why 与 how（2026-06~08）

> 一句话：**LayerNode 不是设计出来的架构，是 Effect 原生 Layer 的四堵墙（不可自省 / 替换无声明 / 无作用域分域 / 错误运行期才炸）在 opencode 的多宿主、双生命周期场景下被撞了两个月后，长出来的 95 行→330 行补丁层。**
> git 考古基线：2026-10-23，dev 分支（HEAD e25b7abf89），全史约 2 万 commits。
> 可信度分级：**【实证】**= PR diff / commit message 可复现；**【分析】**= 动机与代价推演（无书面出处，从代码形态反推）。
> 互链：机制走读见 [../6_modules/0_effect-infra.md](../6_modules/0_effect-infra.md)（本篇讲"为什么、怎么长的"，那篇讲"现在长什么样、怎么用"）；时代背景见 [4_v2-rebuild.md](4_v2-rebuild.md)（V2 重建期，Effect 全家桶豪赌的延伸战线）。

---

## 0. 时间线总览

```
2026-06-09      06-25         06-26          06-28/29      07-03        07-17      07-19/21     08-26        08-27/28      08-31
    │             │             │              │            │            │           │            │            │             │
 #31531        #33937       ecdfff5a42     #34377/85/86  #35175/76   c4830c14    util 抽包     ecbe2fa7    #45705/sdk   36ac35a7
 首版 95 行     tiers 改      hoist 诞生      Replacement   hoist 双修   惰性 deps    layer-node   有效图校验    instance 模块  hoist 删除
 make/group    Tag 雏形      + unbound+tags  定型+测试迁完 (9 秒两连)  解模块环    迁入 util                 抽离+sdk 直连  → shared 参数
 /replace/                                    86 处归零                                                          +图不透明化
 /buildLayer                                                                                                       +CheckTags
 60+ 文件接入
    │             │             │              │            │            │           │            │            │             │
  95 行        ~380 行      拆目录+tree      ——          ——           ——         换宿主包      ——          133 行        330 行(现状)
```

三个主角 commit（后文反复引用，均可 `git show <hash>` 复现）：

| hash | 日期 | 事件 |
|---|---|---|
| `07e5ea9367` | 2026-06-09 | 首版 #31531，95 行，60+ 文件接入 |
| `ecdfff5a42` | 2026-06-26 | #34119，hoist 诞生 + unbound + tags 体系落地 |
| `36ac35a7c8` | 2026-08-31 | hoist 删除，`compile({shared})` 取代 |

---

## 1. 前 LayerNode 时代：defaultLayer 意大利面（git 考古）

【实证】在 `07e5ea9367` 的父提交（`7a54a2c49c`）上检索：

- `git grep -c "defaultLayer" <parent> -- packages/core/src` → **86 处**；
- `git grep -ln "Layer.provide" <parent> -- packages/core/src packages/opencode/src` → 40+ 个文件。

每个服务文件结尾都是同一段"组装咒语"。典型样本（`git show 07e5ea9367^:packages/core/src/git.ts` 第 402 行）：

```ts
export const defaultLayer = layer.pipe(
  Layer.provide(FSUtil.defaultLayer),
  Layer.provide(AppProcess.defaultLayer),
)
```

更上层的组装（`07e5ea9367^:packages/opencode/src/project/project.ts` 第 504-512 行）：

```ts
export const defaultLayer = layer.pipe(
  Layer.provide(EventV2Bridge.defaultLayer),
  Layer.provide(ProjectV2.defaultLayer),
  Layer.provide(ProjectCopy.defaultLayer),
  Layer.provide(AppProcess.defaultLayer),
  Layer.provide(CrossSpawnSpawner.defaultLayer),
  Layer.provide(FSUtil.defaultLayer),
  Layer.provide(Database.defaultLayer),
  Layer.provide(RuntimeFlags.defaultLayer),
)
```

放大看这段代码的三层问题【实证→分析】：

1. **顺序敏感但无检查**。provide 的叠放顺序错了（比如 Database 供在 ProjectV2 依赖之后），类型系统能查一部分（Effect 的 R 参数推断），但"我到底该 provide 哪几层、什么顺序"这件事本身没有声明处，只有每个消费者各自为战。
2. **组合知识重复**。Project 需要 Database + RuntimeFlags + FSUtil……这个知识写在了 Project 的 defaultLayer 里；但测试想要一个"带假 Database 的 Project"时，得把上面 8 行抄一遍、换掉一行——抄漏任何一行，运行期才报。
3. **宿主差异 = fork 整段**。server 要真 LSP，tui 要假 LSP，测试要假 LSP + 假 HTTP——每个宿主一份略异的 pipe 链。V2 重建期宿主数量正在快速增长（见 4_v2-rebuild §0：04 月 server 诞生、06 月 runner/OpenTUI V2 化），这份维护税是按宿主数 × 服务数双重膨胀的。

原生 Effect Layer 时期的形态特征【实证+分析】：

1. **组合即固化**。`Layer.provide(A, B)` 一执行，A 和 B 的关系就被烘进产物，得到的是一棵不透明的 Layer 树。你无法问这棵树"A 的依赖里有没有 Config"，只能读源码。
2. **每个消费者自担组装责任**。86 处 defaultLayer 意味着同一批依赖关系在 86 个地方被手工重抄。漏 provide 一个，错误不在组装处报，而在千里之外的第一次 `yield* SomeService`。
3. **共享靠全局 MemoMap 坉住**。`07e5ea9367^:packages/core/src/effect/runtime.ts` 里已经有 `memoMap`（进程级共享 MemoMap 传入 ManagedRuntime）——团队已经在用 Effect 的原生机制处理"多层组装下同服务只建一份"，但 MemoMap 只认 Layer 实例相等，表达不了"这一类节点都该共享"的语义规则。
4. **测试要复刻整条链**。测试里想换掉某个服务，得手工重新 pipe 一遍它上面的所有 provide——替换无声明机制，只能重排树。

### 1.1 形态翻译对照：pipe 链 → deps 数组（前 vs 后的闭环证据）

【实证】同一段装配的两种写法。左边是 §1 的 git.ts 旧形态，右边是现码 filesystem.ts 的 LayerNode 形态（语义等价的反向翻译）：

```ts
// 原生 Effect（旧，git.ts）                 // LayerNode（现，filesystem.ts）
export const layer =                        export const node = makeLocationNode({
  baseLayer.pipe(                             service: Service,
    Layer.provide(FSUtil.layer),              layer: baseLayer,     ← 本体不变（还是原生 Layer）
    Layer.provide(Location.layer),            deps: [FSUtil.node,   ← 依赖从 pipe 链
    Layer.provide(                                 Location.node,        （组装指令）
      FileSystemSearch.layer),                     FileSystemSearch.node],
  )                                          })                      变成数组（数据）
```

**本质：LayerNode 是在原生 Layer 之上包的一层壳**——`layer` 字段装的还是原生 `Layer.effect` 产物（本体没换），壳做的事是把依赖信息从"埋在 provide 调用序列里的隐式知识"升格为"可检查、可替换、可分作用域的一等数据"。四堵墙逐堵对照这个形态变化：

| 墙 | pipe 链的困境 | deps 数组的出路 |
|---|---|---|
| 不可自省 | 组合产物黑盒，依赖要人肉追链 | 数组即图数据结构，compile 直接 DFS（环检测） |
| 替换无声明 | 换一个服务 = 重写整条链 | `FSUtil.node.replace(替身)` 一次声明，依赖边全改指 |
| 无作用域 | Layer 无全局/每目录概念 | makeLocationNode/makeGlobalNode 的 tag 分域 |
| 运行期才炸 | unbound/环在构建执行时暴露 | compile 静态检查，构建期抛错 |

**精确推论：pipe 没有消失，它退到了"边"上**——deps 只声明"依赖谁"，真正的组装发生在 compile 内部（layer-node.ts:308 `Layer.provide([ambient, ...dependencies])`——provide 还在用，由编译器统一执行）。一句话：**业务代码不再手写 provide 链（声明 deps），编译器替你生成 provide 链（compile 内部）——手工作坊变图编译器**。

两个容易读错的细节【实证】：

1. **产物不是"摊平回一条 pipe 链"**。旧手写形态是每个服务文件里一条扁平的 `baseLayer.pipe(provide(A), provide(B), …)`；compile 生成的不是这个，而是**按图结构递归的嵌套 provide**——每个节点一小段（`node.impl.pipe(provide([ambient, ...build(依赖)]))`，:308），根层再用 `provideMerge` 收拢（:314）。形态跟着图走，不跟手写习惯走。
2. **翻译是惰性的**。pipe/provide 的生成代码住在 `Layer.fromBuild` 回调里（:301）——compile 返回的只是一个"知道怎么把自己翻译成原生 Layer 组合"的描述，真正的翻译发生在**被 build（Runtime 构建）时**。每个 Location 首请求才构建（惰性编译的完整链条：deps 声明 → compile 存翻译器 → 首请求 build 才执行翻译）。

所以同一个 `pipe` 原语的地位变了：旧世界它是**源码里的组装知识**（人写、人维护、错在人）；新世界它是**编译器的代码生成目标**（机器生成、随图重生成、错在编译期暴露）。

值得强调的是：**这不是 Effect 的设计缺陷，而是 Effect 的设计边界**。原生 Layer 的哲学是"组合子 + 类型推断"，它假定你拿到的是最终组装式；它不提供"组装过程本身作为可操作数据"的能力。opencode 需要的恰恰是后者——组装要先于运行被检查、被变换、被多方定制。LayerNode 做的事只有一件：**把 Layer 的组装关系从类型推断层拽到运行期数据层，让 JS 代码可以遍历它**。【分析】

这是 V2 重建进行到第 3 个月时的状态（见 [4_v2-rebuild.md](4_v2-rebuild.md) §时期速览：2026-04-25 core 诞生，06 月 core 服务数量已经涨到几十个）。服务越多，这 4 条特征每一条的代价都在线性放大。首版 #31531 的 author 是 James Long（longster），提交时间 06-09 15:31——core 诞生约 6 周后。也就是说：**他们只忍了 6 周**。这个间隔本身就是数据点——native Layer 在这个规模/宿主矩阵下的痛苦半衰期大约是一个半月。【分析】

---

## 2. 诞生现场：#31531 首版解剖

【实证】`git show 07e5ea9367:packages/core/src/effect/layer-node.ts` 全文 95 行，以下逐段核对了"有什么 / 没有什么"。

### 2.0 首版全骨架（带行号注释的走读）

```ts
// —— 类型层（约 30 行）——
type Node<A, E> = {
  readonly kind: "layer" | "group"          // 注意：没有 "unbound"
  readonly implementation?: Layer.Any
  readonly dependencies: readonly AnyNode[]
  readonly [$OutputType]?: () => A           // 幽灵类型：A/E 只存在于类型层
  readonly [$ErrorType]?: () => E
}
type Missing<Required, Deps> = Exclude<Required, Output<Deps[number]>>
type CheckDependencies<Impl, Deps> =        // deps 输出必须覆盖 implementation 需要的服务
  [Missing<Layer.Services<Impl>, Deps>] extends [never]
    ? unknown : { readonly "Missing dependencies": ... }

// —— 构造层（约 20 行）——
make(implementation, dependencies)          // layer 节点
group(dependencies)                         // 聚合节点 → Layer.mergeAll
replace(source, replacement)                // 声明式替换 + CheckReplacementErrors

// —— 编译层（约 40 行）——
buildLayer(node, { replacements })          // 名字还不叫 compile
  visit(input):
    1. replacements.get(input) ?? input     // 替换先于一切
    2. cache 命中即返回
    3. visiting.has → "Cycle detected: layer#3 -> ..."   // 运行期环检测（自增 id，无名）
    4. group → mergeAll / layer → Layer.provide(impl, deps)
```

三个值得停留的细节：

1. **`$OutputType`/`$ErrorType` 幽灵类型**——A 和 E 从不进入运行期对象（`?: () => A` 且永不赋值），纯靠类型系统携带。这个技巧贯穿至今（现为 class 的 Variance 字段）。
2. **编译器只有 40 行**，但缓存、环检测、替换三条主线齐全。它不是玩具——首版当天就接了 60+ 文件。
3. **API 是函数式的**（`replace(a, b)` 而非 `a.replace(b)`），Node 是裸对象。这两点都要等到 08-31 的不透明化重构才改。

### 2.1 首版就有的（出乎意料地完整）

- **`make(implementation, dependencies)`** —— 给 Layer 挂依赖数组，配 `CheckDependencies` 类型级校验：deps 输出不覆盖 implementation 需要的服务时，报类型错误 `"Missing dependencies"`。首版测试里就有 `// @ts-expect-error Greeting requires Value`（app-graph.test.ts）。
- **`group(dependencies)`** —— 纯聚合节点，`Layer.mergeAll` 编译。
- **`replace(source, replacement)`** —— 声明式替换，且首版就带 `CheckReplacementErrors`：替换引入新错误类型直接类型报错。同测试文件实证：`// @ts-expect-error Replacement cannot introduce a new error`。
- **`buildLayer(node, {replacements})`** —— 编译器：`replacements.get(input) ?? input` 在 visit 入口先替换，**"替换先于物化"**（测试名 `applies overrides before dependency materialization`）——这条语义从第 1 天活到今天。
- **运行期环检测** —— `visiting` 集合 + `stack`，报错带完整路径 `Cycle detected in app graph: layer#3 -> layer#1 -> layer#3`。不可自省墙的第一块砖。
- **编译缓存** —— `cache = new Map<AnyNode, RuntimeLayer>()`，同节点只编译一次。

### 2.2 首版没有的（后来每一件都引发了故事）

| 缺失 | 后果 | 补上的 commit |
|---|---|---|
| **Tag / 作用域** | 只有 layer/group 两种节点，无 global/location 概念 | 06-25 `0e731c2a1a`（tiers）→ 06-26 `ecdfff5a42`（tags） |
| **hoist / shared** | 无共享子图抽取，MemoMap 语义裸奔 | 06-26 hoist 诞生 → 08-31 `36ac35a7c8` 删除换 shared |
| **unbound** | 没有占位节点，测试/宿主无法声明"这个图缺一块，编译时才绑" | 06-26 `2f0343ca24`/`ecdfff5a42` |
| **方法化 replace / 不透明图** | replace 是普通函数；Node 是裸对象字面量，可被结构伪造 | 08-31 `36ac35a7c8`（class + symbol brand） |
| **惰性 deps** | deps 必须静态写出 → ESM 循环 import 直接炸 | 07-17 `c4830c147d` |

**关键读法**：首版 95 行解决的是 §1 的墙 1（半截：有环检测但产物仍可透视）和墙 2、墙 4 的类型层；作用域（墙 3）完全没碰——说明 06-09 时 location 多目录架构还没成型，LayerNode 最初只是"带类型和环检测的 Layer 组装器"。

### 2.3 同批接入的 60+ 文件加的是什么

【实证】`git show --stat 07e5ea9367`：68 文件，+759 行。抽查两个：

- `packages/core/src/git.ts`（+3 行）：`import { LayerNode }` + 一行
  `export const node = LayerNode.make(layer, [FSUtil.node, AppProcess.node])`
- `packages/opencode/src/session/session.ts`（+2 行）：`export const node = LayerNode.make(layer, [BackgroundJob.node, RuntimeFlags.node, Database.node, EventV2Bridge.node])`

即每个服务文件在原有 `layer`/`defaultLayer` 旁**并排**加一个 `node` 导出——旧 defaultLayer 全部保留（86 处是慢慢删的，06-29 `a3776429aa` "finish test layer node conversion"（60 文件）才把测试侧迁完）。这是典型的绞杀者模式：新图先共存、后接管。另有一个 12 行的 `layer-node-platform.ts`，把 NodeFileSystem/HttpClient/LLMClient 等平台依赖也节点化——从第一天起"平台依赖"和"业务服务"就在同一张图里。

---

## 3. Why：四堵墙（每堵配"没有它会怎样"的推演）

前两堵墙的证据已在 §1/§2 铺过，这里集中回答"为什么非造不可"。**推演部分均为【分析】**。

### 墙 1：不可自省——组合产物是黑盒

原生 `Layer.provide` 的产物无法回答"谁依赖谁"。首版 buildLayer 的 DFS + visiting 集合就是冲这堵墙来的：**环在编译期暴露，带完整节点路径**。

**没有它会怎样【分析】**：Effect 对循环 Layer 的报错是运行期 fiber 挂起（build 阶段死等或 defect），且不告诉你环在哪。现状 instance.ts 的图有 ~51 个节点（grep 计数；任务基线 53，口径差在 Location.node/InstancePlugins.node 是否计入）、数百条依赖边，一次误引（比如 Config 依赖了 Agent，Agent 又依赖 Config）要靠人肉读 51 个文件的 deps 数组排查。注意首版连节点名字都没有（只有自增 id `layer#3`）——`name` 字段是 tiers 重构（06-25）时随 Tag 一起引入的，说明团队很快发现"报错可读性"本身就是自省能力的一部分。

自省的红利还有一层【分析】：`hasUnbound(root, node)` 这类图查询（4.4 的条件装配依赖它）只有在"图是数据"时才写得出来。换句话说，墙 1 不只挡住报错，还挡住**一切基于图的自动化**——条件注入、共享抽取、可视化、依赖审计，全是同一能力的不同消费者。这也是为什么首版 95 行里环检测占了将近 20 行：它是自省能力的第一个、也是最痒的一个应用。

### 墙 2：替换无声明——覆盖只能重排树

§1 第 4 条：测试想换一个服务，得重建它上面的整条 provide 链。LayerNode 的答案是 `replace`：**编译时顶替，所有依赖它的边自动指向替身，替身自带新依赖**。opencode 后来全部的定制机制——vanilla 模式、宿主注入、Location 绑定、simulation 假网络——都压在这一个原语上（现状 instance.ts:142-149 的三层 replacements 叠加是集大成）。

**没有它会怎样【分析】**：opencode 的宿主矩阵是 CLI / server / sdk / simulation / desktop（现状 `grep -rln "util/effect/layer-node" packages/*/src` 实证 5+ 个包在用）。每种宿主对同一张图的定制点不同（simulation 连 HttpClient 都换掉，见 07-03 `4790a2772c` 的设计文档："Replacing httpClient…keeps the entire pipeline under test"）。没有声明式替换，N 种宿主 = N 份手工 fork 的组装代码，每加一个服务要改 N 处，漏一处就是某个宿主悄悄用了真实现（对 simulation 来说就是测试打了真网络）。首版测试 `applies overrides before dependency materialization` 写于 06-09——优先级语义（替换先于物化）是第一天就定死的契约。

### 墙 3：无作用域分域——双生命周期表达不了

这是最晚被解决的墙（06-26），也是最本质的。opencode 需要：Database/Config.Global 进程一份；Agent/Tool/FileSystem 每个打开的目录各一份。原生 Layer 没有"这一类节点共享、那一类各建一份"的一等表达——你只能手工把图切成两棵树分别编译（旧 location-layer.ts，151 行，06-26 被删）。

**没有它会怎样【分析】**：二选一灾难——要么 Database 每目录重建（每个目录开一个 SQLite 连接池，"连接风暴"），要么 Agent 全进程共享（目录 A 的会话状态泄漏进目录 B）。手工切树则回到墙 2：每种"切法"是一份 fork，且切错了（location 节点误入全局树）没有检查手段——06-25 tiers 重构引入的 `"Tier X cannot depend on lower tier"` / `"Service S belongs to both tier A and tier B"` 运行期校验（`git show 0e731c2a1a` 可见），就是给这堵墙配的守卫，后来进化为类型级的 `CheckTags`。

### 墙 4：错误运行期才炸——unbound/环/异依赖全在启动后暴露

unbound 节点（只有服务签名没有实现）06-26 才出现，但它补的是首版就欠的账：图里声明"此处需要谁"，编译时无人绑定则 `Unbound layer node: <name>` **启动即抛**，而不是等到使用点。

**没有它会怎样【分析】**：测试替身漏绑一个服务，原生 Layer 的表现是：测试编译通过、运行通过、直到某条冷路径第一次 `yield* ThatService` 才炸——报错位置与漏绑位置隔了整个调用栈。对 51 节点的图，"漏绑"是常态而非意外（现状 core/test/ 下 100+ 测试文件每个都在拼自己的替换表）。unbound + 编译期抛错把这类 bug 的暴露点从"千里之外的使用点"拉回"组装现场"。

---

## 4. How：随事故长出来的特性（PR 编年深读）

### 4.1 06-25 #33937：tiers——作用域的第一刀（实证）

`0e731c2a1a` "support tiered layer nodes"。`Tier` 品牌类型 + `tiers(["a","b"])` 工厂；编译期校验 tier 偏序（上层不得依赖下层）与同服务跨 tier 冲突。95 行涨到 ~380。一天后（06-26）`2f0343ca24`（wip）→ `ecdfff5a42`（#34119）把 tier 泛化成任意命名的 **Tag**，`packages/core/src/effect/node.ts` 落地了沿用至今的拓扑：

```ts
export const tags = LayerNode.tags({ location: ["global"], global: [] })
```

`location` 可依赖 `global`，反向被 `CheckTags` 在类型层禁止。**同一次重构里 hoist 诞生**（见 4.2），layer-node 拆成目录（layer-node.ts 138 行核心 + layer-node-tree.ts 118 行树操作），unbound 节点同期出现，`buildLocationServiceMap` 用 `LayerMap`（idleTimeToLive 60 分钟）实现"每目录惰性编译一份 location 图"。双生命周期架构在此一天成型。

### 4.2 07-03 #35175/#35176：hoist 双修——9 秒钟的两个 commit（实证）

两个 commit 时间戳相差 **9 秒**（11:26:49 / 11:26:58），同一个 bug 的两半：

- `33ed95f09c`（#35175）改 layer-node.ts 一行：hoist 的遍历中 `hoisted.set(node.name, node)` → `hoisted.set(node.name, rewriteReplacementDependencies(node, replacementMap))`；
- `a2769b5ade`（#35176）在调用方 location-services.ts 补 4 行注释，说明为什么替换必须在 hoist 期间应用：

> *Apply replacements during hoist, not afterward: replacements can introduce new tagged dependencies (Location.boundNode depends on Project), and the hoist walk is the only pass that can still slice those back out.*

事故机理【实证+分析】。hoist 时期的编译流水线是**三个串行步骤**：

```
location 图 + 替换表
      │
      ▼ ① bind:      Location.node → Location.boundNode(ref)     （替换表追加一条）
      ▼ ② hoist:     遍历图，把 tag=global 的节点挖出去放共享树   （37 节点图的手术）
      ▼ ③ compile:   剩余图 + 共享树 → Layer                      （另一次遍历，处理替换表）

bug：boundNode(ref) 的依赖里带 Project（global 节点）——但 ② 在 ① 之后跑，
     ② 的挖掘窗口看不到"替换会引入新的 global 依赖"，
     于是 Project 被留在 per-location 图里，每个目录各建一份
```

修复后的正确流水线（#35175/#35176）：② hoist 遍历时**内联应用替换表**（`rewriteReplacementDependencies`）——挖掘与替换必须在同一次遍历里。两个 commit 相差 9 秒：第一个改库（layer-node.ts 一行），第二个改调用方（补 4 行注释固化经验）。这个 9 秒很可能是同一 PR 的两次 push 被分别 merge（#35175/#35176 两个 PR 号），但它标记的状态是真实的：**这个 bug 的"修复"不是增加隔离，而是增加耦合**——hoist 被迫理解替换的语义才能正确工作。两个月后（08-31）的删除是这次耦合的清算：编译器单遍 resolve 里替换查表与共享判定天然同序，问题不再有存在的位置。**隐式图变换越修越错，不如编译器内联显式参数**——这是整篇历史最值得抄走的一句话。【分析】

### 4.3 06-28/29 #34377 → #34386 → #34385：Replacement 形态定型与类型补课（实证）

`84336e4f91` "refine layer node replacements"（06-28）是一次容易被漏掉但影响至今的重构：

- `Replacement` 从对象 `{ source, replacement }` 改为**元组** `readonly [source, replacement]`；替换目标允许两种形态——**Node**（自带依赖边，直接接进图）或**裸 Effect Layer**（必须 closed，`Replacement must be a closed layer` 类型错误守住）；
- `CheckReplacement` 升级为条件类型分发，两条路径都做 `CheckReplacementErrors`；
- location map 的条件装配逻辑重排：只有当图里真的有 `LocationServiceMap` 的 unbound **且没人替换它**时才自动注入默认实现（`hasUnbound` + `hasReplacement` 双查）——"unbound 无人认领就自动补默认"这个模式由此定型。

次日两个 commit 收尾：`fb59606bb4`（#34386，bot 提交，9 行）把 `@ts-expect-error` 断言从包在 never-executed 箭头函数里的写法挪到顶层表达式（包在从不调用的函数里，检查器可能不展开函数体，断言形同虚设——教训通用：**ts-expect-error 必须放在会被检查的位置**）；`a3776429aa`（#34385，60 文件 +728/-602）把 core/opencode 测试整体从手工 provide 链迁到 LayerNode，墙 4 的收益从此落袋：每个测试拼一张小替换表即可获得完整真实图。回看绞杀节奏：06-09 首版（新旧并排）→ 06-29 测试迁完 → 07 月生产路径陆续切——**20 天，86 处 defaultLayer 归零**。08-26 `ecbe2fa7ea` 再补最后一道类型防线："替换不得移除源节点的全部输出 / 必须保留 bundle 的每个输出"（`@ts-expect-error A node replacement cannot remove all source outputs`）。

### 4.4 07-17 c4830c147d：惰性 deps——用图原语解 ESM 环（实证）

`deps` 从 `Items` 变为 `Items | (() => Items)`，Node 的 dependencies 改为 getter。提交信息直说动机："Defer the execution node across the **Session/runner module cycle** until the graph is compiled"。这是模块 import 环（Session ↔ runner 互相引用对方的 node）在编译期才解析的方案——类型校验照常（测试里 `Lazy dependencies must still provide A` 的 ts-expect-error 实证）。值得注意的是他们没有用 DI 容器或运行期 registry 解环，而是给图节点加惰性求值——**图原语又一次吞掉了本属于别的层的问题**。同一时期 07-18 `529d55b1c3` "remove session import cycle" 从模块结构侧配合拆环，双管齐下。

### 4.5 07-19/21：抽 util 包——跨包复用的时刻（实证）

`6ce2519cab`（07-19，Dax）把 layer-node.ts **原样**迁到 `packages/util`（diff 0 行变更，纯移动）；07-21 `e0810753f2`（#37828）整理发布配置，`@opencode-ai/util` 成为可发布包。迁移前的消费者只有 core + opencode；现状 grep 实证消费者 = **cli / core / sdk / server / simulation** 5 个 src + 100+ 测试文件。拉力来源【分析】：simulation（07-03 起替换 HttpClient 做线级仿真）和 sdk 需要同一套图原语，但不应依赖 core 的业务服务——抽象必须下沉。

### 4.6 08-26 ecbe2fa7ea：有效图校验——替换与遍历的一致性（实证）

Kit 的 "validate effective layer graphs"：`hasUnbound` 升级为**带着 replacements 检查"有效图"**（替换后的图，而非声明图）——此前 `hasUnbound(root, X)` 查的是未应用替换的原始图，若替换恰好消除了某个 unbound，旧逻辑会误注入默认实现。同时把 4.4 末提到的"替换输出覆盖"类型防线配平到运行期遍历侧。这是"声明图 vs 有效图"这对概念第一次被显式区分——一个月后 `compile` 的单遍 resolve 正是这个概念的彻底化。

### 4.7 08-27/28 #45705：instance 模块抽离——宿主定制拉力（实证）

`59e7b32c92`（Kit）从 location-services.ts（147 行 → 剩余）抽出 `instance.ts`（133 行）：~50 节点的 graph 数组 + `layer(ref, options)` 编译入口 + **Options 三件套**（plugins / discovery / replacements）。次日 `095e8074a4` "sdk: add scoped direct Session handles" 和 `26fbfa042e` "simplify direct instance wiring" 跟进——sdk 直连会话场景（不开 server、按目录直启实例）是这次抽离的直接拉力。instance.ts 里的 vanilla 机制也在此定型（vanillaReplacements 关掉 discovery 的注释链，见 [0_effect-infra.md](../6_modules/0_effect-infra.md) §5）。

### 4.8 08-31 36ac35a7c8：hoist 之死——自我修正的标本（实证+分析）

Kit Langton 的 "make layer graphs opaque and composable"（127 个文件）。三件事一次做完：

1. **删除 hoist**：`-export function hoist…`（diff 实证整段删除）。等价能力并入 `compile` 的 `CompileOptions.shared`：编译器在 resolve 时凡遇 `graph.tag === shared` 的节点即入共享缓存（`cache.shared` vs `cache.local` 双缓存 + `Layer.makeMemoMapUnsafe` 给非共享部分新鲜 MemoMap）。§4.2 的"替换必须内联进 hoist"问题被结构性消灭：**没有独立图变换步骤了，替换与共享判定在同一次 resolve 遍历里完成**。
2. **图变得不透明**：裸对象字面量 → `GraphValue`/`NodeValue`/`ProviderNode` 类 + `GraphTypeId` symbol brand，`replace` 变方法，防结构伪造。
3. **Tag 依赖规则上类型层**：`CheckTags`——location 节点依赖 location 节点（而非经 global）直接类型报错 `"Invalid tag dependencies"`（06-25 的运行期校验升级为编译期）。

提交信息原文："Replace exposed layer graph assembly with **opaque declarations, checked substitutions, and lifetime-aware compilation**"——这三个词就是现状 layer-node.ts 的自我定义。两月教训浓缩成一句【分析】：**隐式图变换越修越错，不如让编译器内联显式参数。**

现状 compile 的关键段落（layer-node.ts:236-308，供与 0_effect-infra §2 互查）：`resolve()` 单遍完成"替换查表 → group 展开 → unbound 抛错 → 共享判定（tag 命中或继承）→ 同 Layer 异依赖一致性检查"；`Layer.fromBuild` 里 shared 节点用外层 MemoMap、local 节点用 `makeMemoMapUnsafe` 新鲜实例，`Layer.CurrentMemoMap` ambient 保证惰性 LayerMap 继承正确的缓存。首版 buildLayer 的 40 行编译器，最终长成 80 行带双缓存语义的编译器——复杂度没有消失，只是从"多个变换步骤的隐式交互"搬进了"一个步骤的显式参数"。

---

## 5. 现状形态与代价

### 5.1 330 行换来了什么（对照四堵墙逐一验收）

| 墙 | 现状解法（layer-node.ts 行号） | 验收 |
|---|---|---|
| 不可自省 | Definition 可判别联合 + DFS resolve，环报错带节点名（:255-260） | ✅ 编译期、带路径 |
| 替换无声明 | `node.replace()` + 三类型防线（CheckReplacement/CheckErrors/CheckTags，:119-141） | ✅ 类型期拦错误替换 |
| 无作用域分域 | `tags({location:["global"],global:[]})` + `compile({shared})`（:228-229, 248-256） | ✅ 同图双生命周期一次编译 |
| 运行期才炸 | unbound 编译期抛（:271）+ "wired to different dependencies" 一致性检查（:280-286） | ✅ 启动即抛 |

instance.ts:148-159 是全部机制的消费现场：三层 replacements 有序叠加（vanilla → 调用方 → 绑定）→ `compile(graph, {replacements, shared: Node.tags.values.global})` → LocationLifecycle 关停钩子。~51 节点、5 个宿主包、一份组装代码。

用一个具体场景串起四堵墙的解法（现状代码，全部实证）：simulation 想测"provider 返回 429 时 agent 怎么重试"——

```
1. 图里声明 httpClient 节点（platform 层，接进 LLMClient 依赖链）   ← 墙1：依赖关系是数据
2. simulationReplacements: [httpClient.node.replace(fakeNetwork)]    ← 墙2：声明式替换，不用重排树
3. compile(graph, { replacements, shared: globalTag })               ← 墙3：Database 仍全局共享一份
4. 编译瞬间：环/unbound/跨 tag 依赖/tag 不匹配全部检查完毕            ← 墙4：进不了运行期
→ 429 走的是真 SSE 解析路径，测试打不到真网络（07-03 4790a2772c 设计文档原话）
```

### 5.2 付出了什么（诚实清单，分析）

- **学习门槛**：读者要先懂 Effect Layer（本身已是高台阶），再懂 brand 类型、再懂 variance 注解、再读懂 ~100 行纯类型编程（CheckTags 的条件类型嵌套不输任何类型体操库）。`Missing dependencies` 这类错误信息设计得很好，但修错 requires 理解整个类型机器。
- **自建维护成本**：95→330 行的每一步都是本仓库的事故驱动，Effect 升级（他们紧跟 beta，见 4_v2-rebuild）时 Layer 内部 API（`Layer.fromBuild`、`makeMemoMapUnsafe`、`CurrentMemoMap`）变动直接打穿这层——compile 里那些"Preserve the exact output context before Effect appends its own memo-map metadata"注释就是打穿后的疤痕。
- **什么场景不值得**：单宿主、单生命周期、服务 <15 个的应用，原生 `Layer.provide` + MemoMap 完全够——LayerNode 的全部价值来自"同图多实例 × 多宿主 × 双生命周期"的乘积，乘数接近 1 时它是纯开销。它不是通用 DI 最佳实践，是 opencode 形状的最优解。

---

### 5.3 行数预算：复杂度去了哪里

【实证】首版 95 → 现状 330，增量分布（现状文件按段落估读）：

```
类型层（Check*/ReplacementErrors/Tags/LayerOutput…）   ~120 行  ← 4 条类型防线，占增量一半以上
class 化 + brand + ProviderNode                        ~60 行   ← 不透明化（08-31）
compile（双缓存 resolve + fromBuild + buildContext）    ~100 行  ← 吸收了 hoist/bind 的职责
公共 API（make/tags/unbound/group/replace）            ~50 行   ← 与首版基本同量级
```

读法：**增量几乎全部花在"把错误从运行期搬到类型期"和"把多个变换步骤合并成一个编译器"**——服务组装的"快乐路径"从第一天起就没复杂过。这与 §6 的启示 3 互相印证。

---

## 6. 启示：什么时候值得自建框架层

从这段史提炼的触发条件清单（决策模板，供 features/ 卷读者自研特性时对照；**全部为分析**）：

```
自建触发计分（命中 ≥3 再考虑动手）：
 [ ] 同一张图需要编译出多实例（per-location / per-tenant / per-test）
 [ ] 宿主定制矩阵 ≥3（prod / test / simulation / sdk…各换不同子集）
 [ ] 双生命周期域真实存在（有明确的全局资源：连接池、缓存——且有多租户隔离需求）
 [ ] 节点数 >30 或组装代码出现在 >20 处（86 处 defaultLayer 是本案例的引爆点）
 [ ] 你需要对图做静态分析（环检测、依赖审计、可视化）——原生产物黑盒时只能自建
 命中 <3：先写胶水函数熬着；自建的维护税（类型复杂度 + 上游 breaking）是按月付的
```

方法论层面更值钱的三条（均有本案例实证）：

1. **首版只打一堵墙**。95 行首版没有 Tag/hoist/unbound——先让 60 个文件接进来（并排共存，绞杀者模式），让真实使用场景去点菜。四堵墙是被两个月的事故逐堵撞开的，不是白板上规划的四层。
2. **修两次以上的"隐式变换"就该内联**。hoist 修到第二刀（9 秒双修）就该警觉；又熬了两个月才删，删除的收益（shared 参数一处语义）证明犹豫的成本比重构高。
3. **类型防线跟着事故走**。CheckDependencies（首版）→ CheckReplacementErrors（首版）→ CheckTags（8 月才从运行期升级）。每条 ts-expect-error 测试都是一次事故的碑——先运行期报错兜底，模式稳定后再上类型。

与 V2 三豪赌的关系：这是 "Effect 全家桶"赌注的**第二层加注**——不止把运行时押给 Effect，还在其不稳定 API（当时紧跟 beta）之上再造一层 330 行基建。押注逻辑一致：团队判定"装配的正确性"是这类多宿主 agent 平台的核心资产，值得把它从框架手里拿回来自己持有。事后看（2026-10，5 包消费者、instance 一处组装），这一层是 V2 期回报率最高的自建件之一；但回报率前提是 §5.2 那些乘数真的存在。

一个更深的观察【分析】：LayerNode 的演进方向与 React 内部架构史惊人同构——都是"声明数据结构（element/graph）→ 编译器单遍处理（reconciler/resolve）→ 替换与生命周期在编译器内统一"。James Long 的 React 背景（longster，React DevTools 作者）在这里显然不是巧合。当你的问题变成"一组声明的、可替换的、带生命周期的节点如何高效物化"，答案几乎必然走向"写一个小编译器"而不是"组合更多的组合子"。这也解释了为什么 hoist 死了而 compile 活了：**组合子式变换（hoist 是图上的 map）在遇到语义耦合（替换影响共享判定）时会腐烂，编译器式单遍（resolve 一遍定所有）不会**。

---

## 附：关键断言抽查清单（全部可复现）

```bash
cd /home/donghe/github/opencode
git show 07e5ea9367:packages/core/src/effect/layer-node.ts | wc -l        # 95
git show --stat 07e5ea9367 | tail -1                                      # 68 files, +759
git grep -c "defaultLayer" 07e5ea9367^ -- packages/core/src | wc -l        # 86 处（多文件计数合计）
git show 0e731c2a1a:packages/core/src/effect/layer-node.ts | grep -c tier  # tiers 重构在场
git show ecdfff5a42:packages/core/src/effect/layer-node/layer-node-tree.ts | grep -n "export function hoist"  # hoist 诞生 :7
git show 33ed95f09c -- packages/core/src/effect/layer-node.ts             # #35175 一行修
git show a2769b5ade -- packages/core/src/location-services.ts             # #35176 注释 4 行
git log --format="%h %ad %ci %s" --date=short -1 33ed95f09c; git log --format="%ci" -1 a2769b5ade  # 相差 9 秒
git show fb59606bb4                                                       # #34386 测试修正 9 行
git show c4830c147d -- packages/core/src/effect/layer-node.ts             # 惰性 deps
git show 6ce2519cab --stat | grep "layer-node"                            # core→util 零内容变更
git show 36ac35a7c8 -- packages/util/src/effect/layer-node.ts | grep "^-.*function hoist"  # hoist 删除
wc -l packages/util/src/effect/layer-node.ts                              # 330
grep -c "\.node" packages/core/src/instance.ts                            # ~55（nodes 数组 51）
grep -rln "util/effect/layer-node" packages/{cli,sdk,server,simulation}/src | wc -l  # 跨包消费者
```
