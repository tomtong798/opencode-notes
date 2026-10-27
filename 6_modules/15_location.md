# Location 域详解（目录作用域的身份、服务边界与 Project/Workspace/Worktree 三兄弟）

> 一句话：**Location 是 opencode 的目录作用域键——一个 `(directory, workspaceID?)` 二元组决定"哪套服务为你编译"；Project 把目录归到用户可见的项目身份（git remote/root-commit 派生 ID），Worktree 记录项目下的多个 checkout，Workspace 是面向未来远端放置的 provider 挂载位，而 Instance 则是"一个目录 = 一张编译好的 53 节点服务图"。**
> 代码基线：v2.0.30，`dev` 分支（e25b7abf89）；行号实测（2026-10-19）。
> 权威出处：根 `AGENTS.md`「Session Core」条目（Location 作用域约束、implicit-local 放置）。
> [返回 README](README.md) | 前置 [0_effect-infra.md](0_effect-infra.md) §5（LayerNode 组装机制）| 关联 [../5_timeline/3_location-lazy.md](../5_timeline/3_location-lazy.md)（首请求惰性编译时序）——**本篇写域模型本体**：五实体关系、身份派生、缓存/活跃/生命周期语义。

---

## 0.5 通俗完整描述（先读这段再看细节）

**Location = 目录作用域**：opencode 同时服务多个项目目录（你开着的每个仓库），每个目录需要自己的一套服务（它自己的配置、agent、技能、MCP 服务器……）——Location 就是"哪个目录"这个身份，五实体关系口诀：**Location 是键、Instance 是值、Project/Worktree/Workspace 是三本账**。

核心机制是**惰性编译+缓存**：服务启动时不为任何目录建东西；第一个带着某目录的请求到达，才现场编译该目录的完整服务图（读它的配置、发现它的 agent/skill、连它的 MCP——53 个服务），编好放进缓存**永不因空闲过期**（只有显式失效：git 状态结构性变化、空闲回收、运维接口）。这就是"服务秒起、打开项目第一下稍慢、之后一直快"的原因。

三个容易偏差的点：①**缓存不刷新只重建**——配置文件改了不是"热更新缓存里的图"，而是图内部的服务自己监听变更（Config 服务在图内自己有 watcher）；图的失效只用于"目录身份本身变了"。②**全局服务跨目录共享**——数据库、凭据这些（global 作用域节点）全进程一份，编译多个目录图时共享子图不重建。③**Worktree 是 git 工作树账本**（新会话可选在工作树里开——改动隔离），Workspace 是保留位（显式身份目前未启用）。

一句话：**目录=服务图的缓存键：按需编译、永久缓存、全局共享、显式失效——"多项目并行的隔离单元"**。
---

## 0. 模块地图

```
packages/core/src/
├─ location.ts               39 行 ★ Location 服务本体：Ref→Project.resolve→Info
├─ location-service-map.ts   46 行 缓存门面：Service/canonical/reload（LayerMap 包装）
├─ location-services.ts      87 行 ★ buildLocationServiceMap：LayerMap 构建 + 借用绑定
├─ location-activity.ts      96 行 活跃追踪：TTL 60min + 每分钟 sweep → 空闲逐出
├─ location-lifecycle.ts     70 行 单 Location 关停：permission/forms/rpc.close + Shutdown 事件
├─ instance.ts              181 行 ★ 53 节点图 + Instance.layer 编译入口 + Options
├─ instance/
│   └─ service.ts             19 行 Instance.Service：provide(session)（全局图占位）
├─ worktree.ts              357 行 ★ Worktree 服务：create/remove/refresh（策略化）
├─ worktree/
│   ├─ sql.ts                 20 行 worktree 表（project_id+directory 主键）
│   ├─ strategies.ts          74 行 策略注册表（Location 作用域，默认 git）
│   └─ directory.ts           16 行 canonical：resolve+isDir 校验
├─ project.ts               470 行 ★ Project 服务：resolve/activate/sweep（ID 派生）
├─ project/
│   ├─ schema.ts              25 行 ID/Info/Vcs re-export + Vcs struct
│   └─ sql.ts                 63 行 project 表 + upsertProject
├─ workspace.ts             364 行 Workspace 服务：create/provision/connect/destroy
└─ workspace/
    ├─ driver.ts              88 行 WorkspaceDriver 接口 + 注册表（默认空）
    └─ sql.ts                 11 行 workspace 表（id/provider/binding）

公共形状：packages/schema/src/location.ts（Ref/Info/PublicRef，29 行）
```

分工边界（防重复）：**LayerNode 图怎么组装/替换/编译** → [0_effect-infra.md](0_effect-infra.md) §5；**首请求何时触发编译、时序细节** → [../5_timeline/3_location-lazy.md](../5_timeline/3_location-lazy.md)。本篇只答：这些实体**是什么、谁持谁、何时生灭**。

## 1. 心智模型（五实体关系图）

一句话版：**Location 是键，Instance 是值，Project/Worktree/Workspace 是三本账**——前两者本地 git 语义，第三个是远端放置的预留位。

```
进程（全局图：25 节点）
│
├─ LocationServiceMap（全局，LayerMap<Location.Ref, Instance.Services>）
│   │  key: canonical Location.Ref
│   │  value: 整个编译好的 Location 服务图（= Instance）
│   │
│   ├─ "file:///home/me/proj"        ──► Instance A（53 节点：Config/Agent/Tool/...）
│   └─ "file:///home/me/proj-2"      ──► Instance B（独立一份）
│
├─ Project（全局服务，SQLite）
│   │  project 表：id ← git remote / root commit / hg root / 目录哈希
│   │  每个目录 resolve 后归入某个 Project
│   │
│   └─ worktree 表（project_id + directory）
│       └─ 一个 Project 多个 checkout（main + linked worktrees）
│
├─ Worktree（全局服务）── 依赖 LocationServiceMap（load() 时反向借用目录图拿策略）
│
└─ Workspace（全局服务，driver 注册表默认为空 → 当前无远端放置）
    └─ workspace 表：id / provider / binding(JSON) —— 预留

Location.Ref（schema/location.ts:8-11）
  { directory: AbsolutePath, workspaceID?: WorkspaceID }
   └─ 省略 workspaceID = implicit-local（AGENTS.md Session Core 原文）
```

五者关系口诀：

- **Location 持有 Project 的引用**（`Location.Interface.project: {id, directory, canonical}`，location.ts:11-13 + 25-30），不持有 Workspace/Worktree。
- **Instance 持有 Location**（图中第 60 行 `Location.node` 是 53 节点之首）；反过来说，一个编译好的 Instance 就是一个 Location 的全部服务。
- **Project/Worktree/Workspace 都是全局图服务**——它们被多个 Location 共享，所以放全局（tag 方向：location 依赖 global，反向禁止）。
- **Worktree 服务依赖 LocationServiceMap**（worktree.ts:344），形成"全局服务反向借用某目录图"的唯一通路（§8）。

## 2. Location 身份与 canonical 化

### 2.1 Ref 结构与隐式本地

```
packages/schema/src/location.ts:8-11
  Ref = { directory: AbsolutePath, workspaceID?: WorkspaceID }
```

`workspaceID` 全链路可选：省略即 implicit-local placement。AGENTS.md「Session Core」原文：

> "Keep `SessionRunner`, model resolution, tool registry, permissions, and filesystem Location-scoped. Omitted `Location.workspaceID` means implicit-local placement; explicit workspace identity remains reserved for future placement semantics."

即：**当前代码库里 workspaceID 只是身份字段的前瞻**，没有任何代码路径要求它非空；显式 workspace 身份留给未来放置语义（对应 §7 的 Workspace 服务）。`PublicRef`（schema/location.ts:13-15）直接 omit 掉 workspaceID——对外 API 根本不暴露这个维度。

### 2.2 Location 服务的诞生：借 Project 定身份

```
location.ts:19-32 layer(ref)
  ├─ project.resolve(ref.directory)          ← 全局 Project 服务
  └─ Service.of({
       directory: ref.directory,             ← 请求目录（原样）
       workspaceID: ref.workspaceID,
       project: { id, directory, canonical },← resolve 的结果
       vcs: resolved.vcs                     ← {type:"git"|"hg", store}
     })
```

注意 `Resolved`（project.ts:39-46）里有两个目录：`directory`（本次 checkout）与 `canonical`（主 checkout）——从 linked worktree 打开时两者不同，Project 身份锚定在 canonical。

### 2.3 canonical 化（缓存键归一）

```
location-service-map.ts:39-44
  canonical(ref) = Location.Ref.make({
    directory: win32 ? path.normalize(ref.directory) : ref.directory,
    workspaceID: ref.workspaceID,
  })
```

语义要点（注意与任务直觉不同——**没有排序/合并，只有 Windows 路径归一**）：

- 仅 `win32` 做 `path.normalize`（`\` `/` 大小写盘符等价形态统一）；POSIX 原样。
- `AbsolutePath.make` 保证无尾斜杠等基础规整；`/a/b` 与 `/a/b/` 同键。
- **workspaceID 原样保留在键里**——不同 workspaceID = 不同缓存项（即便同目录）。
- 调用点三处统一走 canonical：`location-services.ts:60-62`（get/contextEffect/contextEffectOption）、`location-activity.ts:27`（activity 的 key 还额外拼 `\0` + workspaceID）、`location-lifecycle.ts:35`（事件归属比对）。

0_effect-infra.md §5 已给出 canonical 的"避免等价 placement 变成不同缓存 key"结论；本篇补充：**canonical 是纯函数、无 IO**，因此 LayerMap 键与 Project.resolve 的 fs.resolve（真实路径）是两层不同的归一——前者管缓存等价，后者管身份（符号链接在 resolve 时消解）。

### 2.4 一次 resolve 的实测走样（linked worktree 场景）

```
Location.Ref = { directory: "/repos/proj/.wt/feature-a", workspaceID: undefined }   （用户在 linked worktree 里打开）
  └─ Project.resolve("/repos/proj/.wt/feature-a")
       ├─ fs.resolve → 真实路径
       ├─ fs.up 找到 /repos/proj/.git（worktree 的 .git 是文件，git.repo.discover 识别）
       ├─ id       ← git remote "github.com:me/proj"（Hash.fast("git-remote:github.com/me/proj")）
       ├─ canonical ← git worktree list 中 kind=="main" 的 /repos/proj
       └► Location.Info.project = { id, directory: /repos/proj, canonical: /repos/proj }
           （注意 Info.project.directory 是 repo.worktree 主目录，不是本次打开的 feature-a；
             本次打开目录保留在 Location.directory）
```

缓存键仍是 `/repos/proj/.wt/feature-a`（§2.3），身份键是 `me/proj`（§6.1）——**打开目录、缓存键、项目身份三个"目录"在 linked worktree 场景下两两不同**，这是本域最容易读混的点。

## 3. Instance 服务边界

### 3.1 Instance.Service：全局图里的"借出口"

```
instance/service.ts:10-15
  Interface = { provide: (session) => <A,E,R>(effect) => Effect<A, E|Error, Exclude<R, Services>> }
```

全局图里的 Session 门面拿到 sessionID 后，用 `session.location` 作 key 从 Map 借出整套 Location 服务，再 `Effect.provide` 给执行流——这就是"会话执行借 location 上下文"。实现在 location-services.ts:75-80：

```
Instance.node.replace(Layer.succeed(Instance.Service, {
  provide: (session) => Effect.provide(map.get(session.location)),
}))
```

这个 replace 是 Map **构建时**注入的内层绑定：每个被缓存的 Instance 自己持有一个"委托回本 Map"的 Instance.Service（防递归：自己图里的 Instance.Service 指向 Map 而非自己）。`Instance.node` 本体（service.ts:19）是 `LayerNode.unbound(Service, global)`——unbound 占位，等待 Map 构建时替换（机制详见 0_effect-infra §5.2）。

### 3.2 53 节点按服务域重新分组

instance.ts:59-112 的 nodes 数组（清单时序版见 ../5_timeline/3 §4；此处按**服务域**分组，标注每域的代表消费方）：

| 服务域 | 节点 | 消费场景 |
|---|---|---|
| **身份与生命周期** | Location、LocationLifecycle、Environment | 一切服务的上下文锚点 |
| **配置与指令** | Config、Command、Skill、SkillInstructions、InstructionBuiltIns、InstructionDiscovery、Reference、ReferenceInstructions、InstructionEntry、SessionInstructions | 会话 prompt 组装（6_context.md） |
| **模型与提供者** | Provider、Model、ModelResolver、AISDK、Integration、WebSearch | LLM 调用链（3_model-protocol.md） |
| **插件** | Plugin、PluginHooks、InstancePlugins、PluginSupervisor | 插件宿主 spawn（每个目录独立进程组） |
| **文件系统与终端** | FileSystem、FileSystemSearch、LocationWatcher、FileAccess、FileMutation、Shell、ShellSelect、Pty、Formatter、Vcs | 工具执行的沙箱边界 |
| **权限与工具** | Permission、Tool、ToolOutput、ReadToolFileSystem、McpTool、Mcp | 每目录独立权限会话（11_permission.md） |
| **会话执行** | SessionRunnerLLM、SessionRunnerModel、SessionCompaction、SessionTitle、SessionContext、Snapshot、Form、Generate、Image、Rpc | drain 的全部依赖 |
| **worktree 策略** | WorktreeStrategies | §8：策略注册表按目录配置 |

最后一个是特意的顺序注释（instance.ts:110-111）：

```
// Start repository watches only after boot-critical filesystem and Git work.
LocationWatcher.node,
```

### 3.3 Options 与 vanilla 模式

instance.ts:119-133 的 `Options`：`plugins`（出生自带插件）、`discovery`（默认 true；false = vanilla 启动：不向上扫配置、不读全局配置目录、不加载 plugin-dir——但显式指名的东西照常生效）、`replacements`。vanillaReplacements（:142-145）只换两个节点：`Config.configured({project:false, global:false})` 与 `InstructionDiscovery.configured(...)`。替换链顺序（:150-157）：**vanilla 默认 < 调用者替换 < 实例绑定**（Location.boundNode / InstancePlugins.bound 最后写、最强）。

## 4. LayerMap 缓存语义

### 4.0 组装宿主：buildLocationServiceMap 管的五件事

`buildLocationServiceMap`（location-services.ts:12）不只是"建一个缓存"——它是**图的运营者**：`Instance.layer`（编译器）负责造图，宿主负责图的生老病死。没有它，每次请求都要重新编译 53 节点；有了它，才有"首请求编译、之后复用"的惰性闭环：

```
buildLocationServiceMap
│
├─ ① 缓存与借还（核心）                  :21-22
│    LayerMap.make((ref) => …)
│    └─ ref → Services 按需缓存：
│       首次 get(ref) 触发 builder，
│       之后 get 命中直接借出
│
├─ ② 构建失败的自愈                      :47-49
│    onExit：失败 build 摘除 + invalidate
│    → 环境修复后下次请求重试
│
├─ ③ 失效的清理顺序                      :63-77
│    先摘路由 → inner.invalidate → build.close
│    （uninterruptible；进行中的步骤持有借图
│     到安全边界才交还）
│
├─ ④ 进程退出的善后                      :18
│    owner scope 挂全部图的 finalizer，
│    服务关闭时依次 lifecycle.shutdown
│
└─ ⑤ 自引用解环（最妙的一手）            :79-88
     Instance.node.replace(Layer.succeed(Instance.Service,
       { provide: (session) => Effect.provide(map.get(session.location)) }))
     └─ 宿主把 map 自己注入每张编译出的图：
        图内代码要"某会话所在目录的上下文"时，
        Instance.Service.provide 实际是★回查宿主
        借出那个目录的图——先有鸡还是先有蛋的环，
        由"产物运行时反找宿主借邻居"解开
```

五件事各堵一个洞：① 重复构建（同目录每次请求重编译不可接受）② 失败目录永久坏死 ③ 清理顺序错误会撕断在跑会话 ④ 退出泄漏（插件进程/watcher fd 没人关）⑤ "目录服务"自己也要进图但它又管理所有图的自指矛盾。

位置口诀：**Instance.layer 是造车的工厂，组装宿主是车队管理公司**——车（图）谁造它不管，但哪辆在路上（借出）、哪辆报废（失效）、下班怎么收车（shutdown），全归它。

### 4.1 infinity 的取舍

```
location-services.ts:54-56
  // Retain healthy graphs. Boot failures, not local filesystem probes,
  // decide whether a location (including workspace placement) can retry.
  { idleTimeToLive: Duration.infinity }
```

两层考量（注释即答案）：

- **健康图永不逐出**：逐出意味着 watcher 重建、插件宿主重 spawn、权限会话重置，中断用户工作流——宁可用内存换稳定。
- **"谁有权决定重试"**：只有**编译失败**才摘缓存项（§4.2）；单纯的文件系统探测失败（比如临时权限问题）不算数——`Instance.Error` 类型就是 `FileSystem.DirectoryNotFoundError`（instance.ts:117），instance.ts:160-171 的 catchCause 把非 DirectoryNotFound 的失败全部 `Effect.die`（defect，快速失败不重试）。

### 4.2 失败摘除与 in-flight 竞态

location-services.ts:28-52 的 `onExit` 是缓存层最精细的一段：

```
编译成功:
  lifecycle = Context.get(exit.value, LocationLifecycle.Service)
  if (builds 里该项已被换新) → 这是一次"脱管的在途 boot" → 直接 shutdown
  else → build.close = lifecycle.shutdown（存起来，invalidate 时用）

编译失败:
  if (builds 里该项已被换新) → 空操作（invalidate 触发的重建失败不能逐出自己的替代者）
  else → 从 builds 摘除 + inner.invalidate(ref)（失败即摘，环境修复后下次请求重试）
  失败清理 fork 给 owner scope —— 因为无借用者的 invalidate 会关闭 entry scope
  并 join 当前 lookup fiber，不能在自家身上 await 自己
```

### 4.3 invalidate 何时触发（不是 config 变更）

grep 全库，`invalidate`/`reload` 的触发点只有三处半：

| 触发点 | 位置 | 语义 |
|---|---|---|
| `LocationActivity` sweep | location-activity.ts:81 | 空闲 60min 逐出（§5） |
| VCS 目录切换 | server/src/handlers/vcs.ts:50 | 目录的 git 状态结构性变化 |
| debug 接口 | server/src/handlers/debug.ts:22 | 人工运维手段 |
| `LocationServiceMap.reload` | location-service-map.ts:20-36，由 server/src/handlers/location.ts:23 调 | 先全量 invalidate 再并发重 boot、全结算后报首个失败 |

**注意：config 文件变更不触发 invalidate**——Config 服务自己在 Location 图内部监听配置变化（watcher 域），缓存级 invalidate 只用于"目录身份本身变了"的场景。invalidate 的包装（location-services.ts:63-72）：先摘 builds 路由、`inner.invalidate`，再 `build.close`（= lifecycle.shutdown）；整个 `Effect.uninterruptible`。注释点明借用语义："Running steps retain their borrowed graph until they can hand off at a boundary"——正在跑的会话不会被拦腰斩断，借走的旧图用到安全边界。

## 5. 活跃与生命周期

三个文件三件事，别混：

```
location-activity.ts（全局）── "这个目录还有人用吗？"
  └─ TTL 60min + 每分钟 sweep：过期 → interrupt 该目录的会话 → invalidate

location-lifecycle.ts（Location 图内）── "这个目录怎么体面关停？"
  └─ shutdown：permission.close → forms.close → rpc.close → 广播 Shutdown 事件

location-services.ts builds 账本（全局）── "谁负责按 shutdown 按钮？"
  └─ build.close = lifecycle.shutdown；Layer finalizer / invalidate / 脱管 boot 三条路都会按
```

### 5.1 activity：空闲逐出的完整判定链

location-activity.ts:16-90：

- **touch**（:28-31）：任何带 location 的 durable Session 事件，且该 location 仍在 RcMap 中（还在缓存里）→ 续期 60 分钟。
- **sweep 循环**（:42-85，每分钟）：把缓存里但没有 entry 的 ref 也 touch（新 boot 宽限）；清掉已不在缓存的 entry；对过期项：
  1. 查 `execution.active` 的会话归属该目录 → `interrupt(reason:"inactivity", awaitSettlement:true)`——先停执行、等工具清理结算；
  2. 重查 active：清理期间若**新工作进场**（又有会话落在这个目录）→ 重新 touch、放弃逐出（:73-76）；
  3. 否则删 entry → 日志 `"location services evicted"` → `locations.invalidate`。
- key（:27）用 canonical directory + `\0` + workspaceID，与缓存键同构。

所以"永不过期"（§4.1）与"空闲逐出"并不矛盾：**LayerMap 自身的 idleTimeToLive 是 infinity，逐出决策权在 activity 这一层显式行使**——它能先中断会话再逐出，LayerMap 自己做不到。

### 5.2 lifecycle：单目录的关停序列

location-lifecycle.ts:22-64，Location 图编译即激活：

- 启动时 `project.activate(location.project.id)`（:32）——记录活跃时间（project.ts:259-272，每 Project 每分钟至多一次，不 bump time_updated）。
- 监听 Bus 上的 durable Session 事件，归属本 location 的 → 再 activate（:33-39，fork 出 publish 路径）。
- **shutdown**（:43-58）：幂等（closed 标志 + Deferred）；顺序 `permission.close → forms.close → rpc.close`，最后广播 `LocationEvent.Shutdown`（带本 location 的 Ref，客户端据此知道目录服务没了）；全程 uninterruptible。
- 三条触发路径：① Instance.layer 的 finalizer（instance.ts:172，进程关停/图销毁时）；② Map invalidate 的 `build.close`（§4.3）；③ 脱管在途 boot 的立即回收（§4.2）。

进程退出清理 = 路径①：owner scope（location-services.ts:18）随服务进程关闭，每个缓存图的 finalizer 依次执行 lifecycle.shutdown。

## 6. Project

project.ts（470 行）+ project/ 两文件。核心问题三个：

### 6.1 身份怎么派生（resolve，project.ts:354-397）

```
resolve(directory)
  ├─ fs.resolve（消解符号链接）→ 向上找 .git / .hg（fs.up，markers [:52]）
  ├─ git 分支（:360-381）:
  │   id = remote(url 归一 host/name)  ← 优先：同一远端 = 同一项目
  │       ?? cached(commonDirectory/opencode 文件里存的旧 id)  ← 平滑迁移
  │       ?? rootCommit(首个 root commit hash)  ← 无远端的本地仓库
  │   canonical = gitDirectory==commonDirectory ? worktree
  │              : worktree list 里 kind=="main" 的目录（linked worktree 场景）
  └─ 非 git：hg 用 root changeset（hgRoot :320-339）；都没有 → id = hash("directory:"+dir)
```

设计意图：**clone 两份同一远端 → 同一 Project**（`Resolved.previous` 注释 project.ts:43："the stored project canonical may be another clone"）。`cached()`（:274-280）读 `.git/opencode` 文件里的旧 ID——身份算法升级时旧项目不漂移。

### 6.2 persist 的事务性公告（project.ts:117-219）

resolve 后 upsert project 行 + unarchive + （canonical 消失时）换 canonical；对 vcs 项目再把 canonical 与本次 directory 写进 **worktree 表**（注意：Project 服务自己也在写 worktree 表！），并用 `bus.publish(Worktree.Event.Resolved, ..., {commit: 同事务插行})` 做"事件与行原子提交"（:157-161 注释：崩溃在检查与公告之间 → 下次 resolve 重试而非滞留公告）。还有一段"收养"逻辑（:175-197）：把无 vcs 标记、目录落在新发现仓库内的旧 project 行归并进来。

### 6.3 发现、活跃与归档

- **没有"最近打开"扫描**：Project 列表完全由 resolve 驱动——谁被打开过谁就在表里。`list`（:221-230）按 `time_active desc` 排序，这就是 UI 里"最近项目"的数据来源。
- **activate**：§5.2 所述，lifecycle 启动 + 会话事件都会踩，60s 节流。
- **sweep**（:399-443，sweepNode 每小时跑 :455-470）：canonical 消失且无任何存活 checkout → archive（`time_archived`）；有存活 main checkout → 换 canonical。关键防线 :415-416："Treat unreadable paths as present so transient filesystem errors never archive a Project"——`fs.exists` 失败按"存在"处理，宁可不归档。

project 表（project/sql.ts:12-29）：`id`（PK）/ `worktree`（= canonical 目录）/ `vcs` / 名称图标等 UI 字段 / Timestamps / `time_active`（排序键）/ `sandboxes`。旧表 `project_directory`（:31-47）已 **@deprecated**，指向 worktree/sql 的 WorktreeTable——演进注记 §末。

## 7. Workspace

Workspace 是五实体里唯一"尚未通电"的：`WorkspaceDriver.node = registryNode({})`（driver.ts:88）——**驱动注册表默认为空**，core 不自带任何 provider。

### 7.1 driver 是什么驱动

driver.ts:26-62 的 `Interface` 四方法，面向"远端/云端执行环境 provider"（沙箱、容器、VM…）：

```
create(workspaceID)       → binding     幂等：同 ID 重试/崩溃后重试/他进程竞争 → 认领既有资源
connect(id, binding, save) → EnvironmentDriver.Driver（带 Scope）
suspendForIdle(...)        → 挂起（省资源保状态）
destroy(id, binding|null)  → binding 为 null 也要找漏网资源清理（create 成功但 binding 未落盘的窗口）
```

`Binding`（:14）是"provider 私有的最小 JSON 重连凭据"——core 只存不读（workspace/sql.ts:8 的 `binding text json` 列）。`connect` 返回 `EnvironmentDriver.Driver`——即环境驱动接口（进程 spawn + 文件操作），这是 Workspace 与 Environment 域的接缝。

### 7.2 Workspace 服务的状态机（workspace.ts）

| 操作 | 行号 | 语义 |
|---|---|---|
| `create` | :225-259 | 纯落库（binding=null），零 provider 工作；同 id 同 provider 幂等，异 provider → CreateConflict |
| `provision` | :116-149 | 共享 attempt（Deferred）：首个调用者持 KeyedMutex 做 `driver.create → saveBinding`，并发者 join；destroy 可取消在途 attempt |
| `connect` | :261-318 | acquire/release 引用计数（active Ref），唤醒或复用连接；返回 `spawner` + `overrides`（Files 七操作都走"唤醒→用→释放"） |
| 空闲挂起 | :191-222 | 每分钟轮询：无 active 且 lastActivity 超 20min → `driver.suspendForIdle`，关连接 scope；注释明言"故意持锁挂起"防与 spawn 竞态 |
| `destroy` | :319-351 | 结算 attempt → 关连接 → driver.destroy（null binding 也调用，防资源泄漏）→ 删行 |

文末三条 TODO（:358-363）值得抄——boot janitor、可中断冷唤醒、"keyed shared-attempt 助手该抽出来了（rc.111 的 RcMap/Cache 都表达不了）"。这印证 Workspace 是进行中的 workspace-plan 系列。

### 7.3 与 Location 的关系（现状）

workspace 表与 Location.Ref.workspaceID 是**同一预留的两半**：schema 侧留了字段（§2.1），core 侧留了服务与表，但 `registryNode({})` 空 + 无调用方按 workspaceID 走 workspace 服务——explicit workspace placement 尚未接通。当前所有 Location 都是 implicit-local。

## 8. Worktree

worktree.ts（357 行）+ worktree/ 三文件。**策略化**是本域关键词：git 只是默认策略，插件可注册别的（strategies.ts:47-53，`Editor.add` 会把新策略设为 selected）。

### 8.1 表与策略

worktree 表（worktree/sql.ts）：`(project_id, directory)` 复合主键 + `strategy`（null = 无主/发现根，"git" 等 = 托管子树）。策略注册表是 **Location 作用域**（strategies.ts:66-73 `makeLocationNode`）——每目录可配置不同策略与落地目录（默认 `global.data/worktree/<project-id[:6]>/`，:45）。这就是全局 Worktree 服务要**反向借用 LocationServiceMap** 的原因：

```
worktree.ts:121-128 load(directory)
  ├─ fs.isDir 校验
  ├─ 命中当前图（current.directory === directory）→ 直接用
  └─ 否则 locations.contextEffect(Location.Ref.make({ directory }))
     → 等插件激活 → 取那个目录图里的 WorktreeStrategies.Service
```

（Desktop 提交流程的 `worktreeSelection==="create"` 走的就是本节 create，见 ../4_desktop/1_multi-process.md。）

### 8.2 create：复用与防冲突（worktree.ts:204-255）

```
project(projectID) → 项目行
load(row.worktree) → 策略设置（selected 策略 + 落地目录）
source(...)：from 目录 canonical 后必须在 worktree 表内（SourceDirectoryNotFoundError）
makeDirectory 落地目录 → name = input.name ?? Slug.create()
  └─ 防冲突循环：path.join(directory, name) 存在 → name-2 … name-10，超过 → DestinationExistsError
selected.create({directory, sourceDirectory, branch}) → 策略落盘（git worktree add 等）
ops.create 落表（replace:true——显式创建可夺无主行）
row.commands.start 存在 → 在新目录跑启动命令（env: OPENCODE_WORKTREE_BASE/PATH）
```

### 8.3 remove / refresh（对账）

- **remove**（:257-271）：只允许删**有 strategy 的托管行**（`InvalidDirectoryError` 兜底，:261）；策略自删（git worktree remove --force?）再删行。
- **refresh**（:273-327）：全量对账——现存行逐个 `isDir`；以"canonical + 所有存活无 strategy 行"为根，逐策略 `list()` 枚举托管子树（失败仅告警降级 :297-301）；发现新目录补行（发现可认领无主行但**不抢他人 strategy**，ops.create 的 setWhere :168）；消失的目录删行；事务内结算后统一发 `Event.Updated`。

## 9. 不变量与设计取舍

1. **tag 方向不变量**：Location 图依赖全局图，反向禁止（编译期保证，见 ../5_timeline/3 §0）。因此 Project/Worktree/Workspace 必须是全局服务，Location 图内的 WorktreeStrategies 只能是配置态。
2. **implicit-local 不变量**（AGENTS.md 原文见 §2.1）：workspaceID 可选且当前无人赋值；PublicRef omit 掉它——外部世界今天只有本地放置。
3. **缓存永不过期 + 显式逐出**：健康图由 LayerMap infinity 保命；逐出唯一正当理由是"确认没人用了"（activity），且必须先停会话、结算工具、再摘路由（§5.1）。**取舍**：内存占用随打开过的目录线性增长，换执行稳定。
4. **失败不污染缓存**：boot 失败即摘；文件系统探测失败不算 boot 失败（§4.1 注释）；非 DirectoryNotFound 的 boot 失败是 defect 直接 die。
5. **借用图跨 invalidate 存活**：invalidate 只断路由，正在跑的 step 持旧图到安全边界（§4.3）——正确性优先于立即回收。
6. **Project 身份锚远端**：remote > 缓存旧 ID > root commit > 目录哈希（§6.1），clone 漂移被身份算法吸收；sweep 宁可漏归档不可误归档（unreadable = present）。
7. **canonical 双层**：缓存键 canonical（纯函数，win32 归一）≠ 身份 canonical（fs.resolve 消解链接 + git main worktree）。两者都叫 canonical 但职责不同，读代码时别混。
8. **Worktree strategy 的所有权纪律**：发现只认领无主行、绝不覆盖他策略（ops.create setWhere）；remove 只删托管行——防误删用户手工 checkout。

## 10. 失败与恢复

| 场景 | 行为 | 出处 |
|---|---|---|
| 目录不存在（首次编译） | `FileSystem.DirectoryNotFoundError` 作为**可预期错误**失败（非 defect），缓存摘除，下次请求重试 | instance.ts:160-171 + location-services.ts:43 |
| 目录编译中崩了别的缺陷 | `Effect.die` 快速失败；失败摘除 fork 给 owner 清理 | instance.ts:167 + :47-48 |
| 目录在服务活着时被删 | LayerMap 不主动探测；activity 60min 后逐出；Project sweep 1h 后 archive（unreadable 视为 present，瞬时错误不误伤） | §5.1 + project.ts:415-416 |
| worktree 被外部删除 | Worktree.refresh 对账删行；Project sweep 找存活 main checkout 换 canonical 或 archive | worktree.ts:315-324 + project.ts:420-433 |
| 从消失的旧 canonical 重新打开 | resolve 换 canonical（仅当旧的确实不在了 :132-144）+ unarchive | project.ts:125-144 |
| provision 崩在 create 与落盘之间 | destroy 以 null binding 调 driver，provider 按 workspaceID 认领漏网资源 | driver.ts:52-57 + workspace.ts:337-346 |
| invalidate 与在途 boot 竞态 | builds 账本代际比对：换新者不逐出自己的替代者；脱管 boot 立即 shutdown | location-services.ts:33-41 |
| 逐出清理期间新工作进场 | 重查 active，有新会话则 re-touch 放弃逐出 | location-activity.ts:71-76 |

## 演进注记

- **旧笔记 4.11（project-location-worktree）vs 现码**：现仓库中 Project 身份已完整落地为"remote > cached > rootCommit > hash"四级派生，且 worktree 目录簿从 project_directory 表（已 @deprecated，project/sql.ts:31）迁到独立 `worktree` 表 + 策略列；旧"project-location-worktree"三分笔记描述的静态归属模型，现在多了 strategy/refresh 对账与收养（adopted）逻辑。
- **LocationServiceMap 从 TTL 到 infinity**：早期 60min TTL（旧笔记时序篇记载过），现 `idleTimeToLive: Duration.infinity` + activity 显式逐出——逐出决策从缓存层上移到业务层，因为要先停会话。
- **Workspace 整域为新增**（workspace-plan 系列，文末 3 条 TODO）：driver 注册表空转，explicit placement 未接通；`Location.workspaceID` 字段、workspace 表、driver 接口都是为它预埋。
- **instance.ts 节点数**：53 个（../5_timeline/3 实测），本篇按服务域重分组（§3.2），两篇清单一致。
- **project.activate 节流**：60s/Project（project.ts:60 `ACTIVATE_INTERVAL`），且 activate 不 bump time_updated（:265-266 注释"Activity is not a metadata edit"）——事件语义的洁癖值得学。

## 相关篇

- [0_effect-infra.md](0_effect-infra.md) §5 —— LayerNode 组装机制、Instance.node 占位与替换（本篇 §3.1 的机制侧）
- [../5_timeline/3_location-lazy.md](../5_timeline/3_location-lazy.md) —— 首请求惰性编译时序、53 节点清单时序版
- [4_session.md](4_session.md) —— SessionExecution 如何经 `LocationServiceMap.get(session.location)` 发现放置
- [1_data-layer.md](1_data-layer.md) —— project/worktree/workspace 三张表所在的 SQLite 层
- [../4_desktop/1_multi-process.md](../4_desktop/1_multi-process.md) —— Desktop 的 worktreeSelection==="create" 流程（消费 §8.2）
