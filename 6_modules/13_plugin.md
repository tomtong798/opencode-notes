# 插件系统详解（宿主机制：干预点体系 · 同进程装载 · 代际热重载）

> 一句话：**插件系统是 opencode 的统一扩展内核——所有外部代码（以及绝大多数内置功能）都以"插件"身份在同进程内注入一张按 domain.name 寻址的 hook 回调表，由 supervisor 驱动的代际（generation）机制实现装载、前缀 diff 热重载与失败回退。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19）。
> 权威出处：`packages/plugin/README.md`、`packages/plugin/src/README.md`、`packages/plugin/src/effect/PLAN.md`
> [返回 README](README.md) | 前置 [0_effect-infra.md](0_effect-infra.md) | 关联 [2_config.md](2_config.md)（config/plugin/）· [8_agent.md](8_agent.md)（agent 注册）· [9_tool.md](9_tool.md)（工具注册）

---

## 0.5 通俗完整描述（先读这段再看细节）

**插件 = 外部代码注入干预点**：宿主在执行流的关键位置埋了 18 个钩子（提交前/工具执行前后/权限评估时…），插件注册到这些钩子上就能观察或改变行为——比如"每次工具执行前记个账"、"改写提交内容"。

三个澄清直觉的要点：①**插件是同进程的**——不是独立进程沙箱，它跑在你的进程里直接调宿主服务（所以能力巨大、信任要求也高；加载器会拦截插件自带重复依赖保证 Effect 单副本）；②**内部功能也是插件**——15 个内置工具、7 个 agent 全部以普通插件身份注册（无特权通道），"配置插件"读你的 opencode.json 把自定义内容灌进注册表——**插件系统是 opencode 的扩展底座而不只是外挂接口**；③**干预点有类型纪律**——18 个钩子只有"工具执行前"允许失败拦截（拒绝这次调用），其余只能观察或转换，失败通道在类型层锁死。

生命周期：宿主装配时从四来源（内置/npm 包/本地目录/配置声明）解析模块 → 生成代次（generation）→ 订阅配置变更热重载。插件 API 的门面（ctx）提供 tool/agent/skill/command 的 transform 编辑器——注册即编辑注册表。

一句话：**同进程钩子宿主+注册表编辑器——18 个埋点观察/改变执行流，内部外部同一通道，扩展即插件**。
---

## 0. 模块地图

两个包分工：`packages/plugin` 是**插件 SDK**（插件作者 import 的 API 面 + 宿主侧装载器，发布为 npm 包 `@opencode/plugin`）；`packages/core/src/plugin/` 是**宿主装配**（把 SDK 的装载器接到 config/npm/watcher/registry 上）。

```
packages/plugin/src/                ★ SDK 本体（发布为 @opencode/plugin）
├─ host.ts        Host.resolve/load：入口点解析（server/tui/rpc）+ import（48 行）
├─ source.ts      createPluginSources：本地源指纹缓存 + watch 钩子（66 行）
├─ source.bun.ts  Bun 装载器：require.cache 失效 + 依赖图扫描（71 行）
├─ source.node.ts Node 装载器：module registerHooks + URL 查询参数换血（52 行）
├─ source.package.ts npm 包缺依赖时的 watch 目标推导（13 行）
├─ runtime.ts     Bun.plugin 运行时模块接管：effect/@opencode/plugin 单副本（52 行）
├─ runtime-modules.ts  宿主提供的模块清单（113 行）
├─ rpc.ts         纯 re-export @opencode/schema/rpc（1 行）
├─ permission.ts / storage.ts / worktree.ts / app.ts / options.ts  小适配件
├─ effect/        ★ Effect 版插件 API（domain 类型面，817 行 / 22 文件）
│   ├─ plugin.ts     Context/Plugin/define（64 行）
│   ├─ session.ts    SessionHooks 12 个事件类型（172 行）
│   ├─ tool.ts       ToolHooks + ToolFailures（60 行）
│   └─ registration.ts Hooks/Transform/ModelHookOptions（27 行）
├─ promise/       Promise 版插件 API（setup/async 回调适配层，24 文件）
└─ tui/           终端 UI 插件 API（context/plugin/solid）

packages/core/src/plugin/           ★ 宿主侧（23 文件，3,375 行中约 2,600 行属本篇）
├─ plugin.ts      ★ Plugin registry：activate 前缀 diff + slot + Latch 就绪门（300 行）
├─ plugin/service.ts  Service tag + awaitActivation（34 行，独立文件避免循环依赖）
├─ hooks.ts       ★ PluginHooks：domain.name 回调表 + trigger 串行链（108 行）
├─ host.ts        ★ PluginHost.make：组装 Plugin.Context（23 个服务 → ctx 各域，702 行）
├─ module.ts      ★ PluginModule：npm 安装/策略门/入口解析/源装载（151 行）
├─ supervisor.ts  ★ PluginSupervisor：变更流→resolve→registry.activate（268 行）
├─ internal.ts    ★ 内部插件清单 pre/post + guarded ID（290 行）
├─ instance.ts    InstancePlugins：实例出生绑定的插件（45 行）
├─ sdk.ts         SdkPlugins：embedder 宿主全局贡献通道（34 行）
├─ update.ts      PluginUpdate：npm 过期检查/更新（74 行）
├─ agent.ts / command.ts / skill.ts / provider.ts / plan.ts ...
│                 内部插件个体（agent 注册细节归 8_agent.md，此处只作清单项）
└─ identity.ts / mcp-codemode-defaults.ts / models-dev.ts / warming.ts ...  同上
```

互链去重：`internal.ts:216-250` 的 pre/post 清单中每个工具/agent 插件的**注册内容**分别归 9_tool.md / 8_agent.md；`config/plugin/`（Operation 语法、config 扫描）归 2_config.md。本篇写它们共同 riding 的**宿主机制**。

## 1. 心智模型

插件 = 一段在外部定义、被宿主同进程执行的 `effect(ctx)` 函数。它拿到的 `ctx` 是宿主全部能力的受限视图（读 API + transform 编辑器 + hook 注册器）。宿主不区分"内置功能"与"第三方扩展"——两者走同一条 `activate` 通道，这是 v2 最核心的设计决定（见 §5、§8）。

```
  config 声明 / SDK 注册 / 内部清单          npm / 本地文件
  ┌──────────────┐   ┌──────────────┐   ┌───────────────────┐
  │ ConfigPlugin │   │ PluginModule │←──│ Npm.add / watcher │
  │ Source.ops() │→ │  .load()     │   └───────────────────┘
  └──────┬───────┘   └──────┬──────┘
         │                  │ Generation {id, revision, effect}
         ▼                  ▼
  ┌─────────────────────────────────┐
  │ PluginSupervisor.resolve()      │ 按 pre→package→post 排序、去重、记失败
  └──────────────┬──────────────────┘
                 ▼
  ┌─────────────────────────────────┐  前缀 diff：unchanged prefix 存活
  │ Plugin.activate (registry)      │  变更槽位：close 旧 scope → load 新
  └──────────────┬──────────────────┘
                 ▼  plugin.effect({...host, storage})
  ┌─────────────────────────────────┐
  │ ctx.agent.transform(...)        │→ 容器写入（State.batch 内）
  │ ctx.session.hook("prompt", cb)  │→ PluginHooks 回调表
  └─────────────────────────────────┘
                 ▼  运行时
  SessionPrompt / model-request / Tool / Permission
        → hooks.trigger("session","prompt",event) → 串行执行回调 → 改 event
```

三个关键词：
- **干预点（hook）**：运行时拦截，回调收到可变 event 对象，改完往下传（§2）。
- **transform**：状态域（agent/model/tool/...）的批量重建编辑器，注册期一次性运行（§6）。
- **代际（generation）**：每次变更全量重算插件有序表，registry 做前缀 diff 决定谁卸谁装（§7）。

## 2. 干预点体系（hooks 全清单）

权威定义在 `packages/plugin/src/effect/session.ts:110-124`（SessionHooks）等各 domain 文件；调度器是 `core/src/plugin/hooks.ts`。全表（★=可改 input+output 双向；○=只读观察；▣=可短路）：

| domain | hook 事件 | 事件形状（可变字段） | 拦截语义 |
|---|---|---|---|
| session | `prompt` | `session.ts:15-22`：`prompt`（DeepMutable PromptInput）、`metadata` | ★ 改用户输入（structuredClone 后注入，`session/prompt.ts:40-44`） |
| session | `context` | `:39-45`：`system[]`、`messages[]`、`options`、`tools{}` | ★ 模型派发前最后改写点（README："mutable immediately before provider dispatch"） |
| session | `compaction` | `:47-53`：继承 Context + `result?` | ★ 设 `result` 可**跳过模型调用**直接给定摘要 |
| session | `generate` | `:55` 同 Context | ★ |
| session | `title` | `:57-60`：`result?` | ★ 设 `result` 跳过标题模型调用 |
| session | `model.request` | `:74-81`：`baseURL?`、`headers` | ★ 改端点/请求头（`session/model-request.ts:305` 触发） |
| session | `http.request` | `:83-90`：`request`（fetch Request） | ★ 逐请求改写（`model-request.ts:343`） |
| session | `http.response` | `:92-99`：`response` | ★ |
| session | `experimental.ws.handshake/send/receive` | `:101-141`：`url/headers`、`frame` | ★ WebSocket 三点（实验性，注释明示协议改写风险自负） |
| session | `retry` | `:143-151`：`decision` | ★ 改重试决策（`model-request.ts:399`） |
| tool | `execute.before` | `effect/tool.ts:24-33`：`input` | ▣★ **唯一允许 fail 的 hook**：抛 `Tool.Error` 在工具运行前拒绝调用（`tool.ts:53-56` 注释 + hooks.ts:23） |
| tool | `execute.after` | `:34-52`：`result` / `error` | ★ 改输出或错误（不可 fail） |
| permission | `evaluate` | `effect/permission.ts:11-20`：`effect`、`message` | ★ 改裁决（allow/deny/ask，`permission.ts:179` 触发） |
| shell | `create.before` | `effect/shell.ts:5-10`：`command/cwd/timeout/shell/env` | ★ 改 shell 命令 |
| aisdk | `sdk` | `effect/aisdk.ts:5-12`：`sdk?` | ★ 替换 SDK 工厂结果（host.ts:158-173 按 providerID 预过滤） |
| aisdk | `language` | `:13-19`：`language?` | ★ 替换 LanguageModelV3 实例 |

失败通道在类型层锁死：`hooks.ts:24-30` 的 `Failures` 接口——除 `tool.execute.before: Tool.Error` 外全部 `never`（`NoFailures<Spec>` 映射）。这意味着 hook 抛错是 **defect**，走 §9 的隔离/禁用路径而不是调用方错误路径。

调度语义（`hooks.ts:88-95` trigger）：
- **同 domain.name 内严格串行**，按注册顺序（即插件装载顺序）执行；
- 每个 callback 直接改同一个 event 对象引用 → 后注册者看到前者的修改（链式 pipeline）；
- **无超时**：一个 hook 挂起就挂起整个触发点（取舍见 §8）；
- `ModelHookOptions.providerID`（`effect/registration.ts:22-24`）支持按 provider 过滤；trigger 侧 `hooks.ts:35-40` 从 event.model 提取 providerID 匹配，不匹配则跳过该回调。

注册 API 返回 `Registration{dispose}`（`registration.ts:5-7`），dispose 从回调表移除（`hooks.ts:76-83`），scope finalizer 兜底。

**hook vs event.subscribe 的职责分界**（插件的两条消费通道，容易混淆）：hook 是"**改**"的通道——同步、在执行路径上、入参即该位置流动对象的 DeepMutable 版本（改完回到执行流）；`ctx.event.subscribe()` 是"**看**"的通道——异步旁观 Bus 全量事件流（如 `session.text.delta` 逐 token、`inbox.enqueued` 提交后），不在执行路径上、不能改任何东西，想反应就自己调 ctx 的动作 API。选择标准一句话：**要在事前/事后改变行为用 hook，要全程观察用 event**。

## 3. 宿主装配

### 3.1 插件从哪来（四个来源通道）

`supervisor.ts:152-161` 装配时的完整 boot 序：

```
pre  = internal.pre（标记 revision:"internal", source:"builtin"）
     + SdkPlugins.all()   （embedder 宿主全局，source:"sdk"）
     + InstancePlugins.all()（实例出生参数，source:"sdk"）
post = internal.post
packages = ConfigPluginSource.operations() 经 module.load 的产物
```

- **config 声明**：`config/plugin/source.ts:115-119` 把 config 字符串解析为 Operation——`"pkg"` → `{type:"add"}`、`"-pkg"` → `{type:"remove"}`（支持 `*` 与 `prefix.*` 通配，`supervisor.ts:27-28` matches）。
- **npm 包**：`module.ts:103-109`——非绝对路径先过 `ManagedPolicy` 策略门（`module.ts:52-59`，`integration.use` deny 则返回 `{blocked}`），然后 `npm.add(target)`（或 `install:false` 时仅 resolve）。
- **本地文件/目录**：绝对路径直接走 `sources.read`（`module.ts:120-122`）；单文件 server 入口走 `pathToFileURL`（`module.ts:112-115`）。
- **入口解析**：`plugin/src/host.ts:17-44` `Host.resolve` 探测 `server`/`tui`/`rpc` 三个子入口（package exports 或目录下同名文件）， tolerant 处理六种 MODULE_NOT_FOUND 错误码。

装载产物是 `Generation`（`core/src/plugin/service.ts:23-27`）：`{id, effect, revision, source?, features?}`。模块 default 导出校验（`module.ts:78-91`）：必须有 `id` + `effect`（Effect 版）或 `setup`（Promise 版，经 `PluginPromise.fromPromise` 适配，`module.ts:134`）。

### 3.2 装载顺序与重排

`supervisor.ts:99-103`：`[...pre 过滤 enabled, ...packages 过滤 enabled, ...post 过滤 enabled]`。注册表激活遇重复 ID 会 die 并**连内置插件一起丢掉整代**（`supervisor.ts:104-105` 注释），所以 resolve 层先把后出现的重复者转为 failure 条目（`supervisor.ts:106-118`）而非让激活崩溃。

### 3.3 错误隔离（一个插件崩了会不会拖垮宿主？）

分层隔离，答案是不会，但代价分级：

1. **装载失败**（npm 解析/入口缺失/schema 不符）：`supervisor.ts:59-67` catchCause → 记入 `failures`（带 `err_xxxxxxxx` ref），宿主继续；该 target **保留上一代运行中的版本**（`supervisor.ts:85-90`："The new revision never became a generation, so the one already running keeps its place"）。
2. **setup 失败**（effect 抛错）：`core/src/plugin.ts:81-86` 记 warning，slot 标 error；若上一代同 id 健康，**回退装载旧版**（`plugin.ts:156-166`："fallback"），旧版也装不上才彻底下线（`plugin.ts:167-169`）。
3. **transform 运行期失败**：`plugin.ts:50-63` State.group 捕获 → 禁用该插件 + 排队刷新受污染的状态域（`plugin.ts:182-236` pendingFailures 消费者），宿主与其余插件不受影响。
4. **hook 运行期失败**：defect 沿触发点传播——不拖垮其他插件，但当前 session step 失败（§9）。

## 4. 进程模型与 RPC

**同进程、无独立插件进程。** 这是实测结论：装载即 `importModule(entrypoint)`（`plugin/src/host.ts:46-48`），插件 effect 直接在宿主 Effect runtime 里跑。`packages/plugin/src/rpc.ts` 只有 1 行——`export { Rpc } from "@opencode/schema/rpc"`，是 RPC **契约类型**（插件与宿主共享的 schema 定义，供 rpc entrypoint 的插件消费），不是进程间通道。

同进程的代价靠**运行时模块接管**解决。`plugin/src/runtime.ts:12-38`：Bun 环境下注册 `Bun.plugin("opencode-plugin-runtime")`，把 `effect`、`@opencode/plugin` 等specifier 重定向到宿主实例（`build.module`），并**主动拦截**插件自带 node_modules 里的这些包（`runtime.ts:28-34` onLoad 抛错："plugins must use the host's effect and @opencode/plugin modules"）。README 对应条款：effect 必须声明 peerDependency、不得 bundle。理由：两份 effect 不共享 fiber/logger/Schema 内部。`createForeignPackageFilter`（`runtime.ts:40-52`）用正则区分宿主根与插件 node_modules。

**热重载的换血机制**（SDK 侧最精巧的部分）：
- **Bun**（`source.bun.ts:12-70`）：用 `Bun.Transpiler.scan` 递归扫描入口的本地依赖图，逐文件 `delete require.cache`（Bun 的 ESM 也走此 cache，`:21-22` 注释）+ track 每个 file 供 watch；version 是单调递增的 `++generation`。
- **Node**（`source.node.ts:10-51`）：`registerHooks` 自定义 resolve——仅当 parent URL 带本次 `__opencode_reload=<version>` 查询参数时介入，把解析到的文件 URL 加上同样的参数 → 缓存键不同 = 强制重新求值。
- **指纹缓存**（`source.ts:13-18`）：reload 前对每个 track 过的文件算 sha256（目录用排序后的 readdir JSON，`:53-58`），全部未变则直接返回上次结果——"unchanged failing modules must not repeat import-time effects"（`:35-36` 注释）。

`module.ts:26-42` 宿主侧把 watch 接到 `filesystem/watcher`：变更文件不存在时递归 watch 最近存在的祖先目录（`module.ts:66-76`），使"先删目录再建"也能触发恢复。

## 5. 内部插件机制（无特权通道）

`internal.ts` 的设计：**内置功能不走后门，走插件协议本身**。`internal.ts:276-290` `list()` 把 pre/post 两个清单（`:216-250` / `:252-270`）包装成普通 `Plugin`——`effect: (host) => plugin.effect(host).pipe(Effect.provide(context))`。区别只有两点：

1. **context 预注入**：内部插件 requirements 是全部 48 个核心服务（`:108-158` services 清单），`list()` 用 `Context.pick(...services)`（`:278`）捕获后经 `Effect.provide` 塞给它们——外部插件的 ctx 只有 host.ts 的公共面，拿不到 `Config.Service`、`FileAccess.Service` 等内部服务。这是**能力差**而非**通道差**：注册、transform、hook 走完全相同的表。
2. **revision 恒为 `"internal"`、source 为 `"builtin"`**（`supervisor.ts:153`）：代际 diff 中 (id, revision) 不变 → 永不重载（对照 `instance.ts:33-37` 的同款注释："The constant revision is load-bearing"）。

`guarded` 集合（`internal.ts:272-274`）：`opencode` provider 插件与 `ConfigPolicyPlugin` 不可被 config remove——"Repository config must not switch off policy enforcement or the Console connection"。`supervisor.ts:43` resolve 时跳过对 guarded ID 的删除。

pre/post 的顺序语义：pre 里是基础设施（工具、agent、provider、vcs...），post 里全部是 `Config*Plugin`——config 注入必须最后跑，覆盖 pre 阶段容器写入。9_tool.md 已列工具侧清单，此处只记骨架。

身份示例（最小内部插件）：`identity.ts:18-32` `IdentityPlugin`——hook `context/compaction/generate` 三事件，向 `event.system` splice 一段"# Your Model"系统部件。这是理解 hook 模式的 15 行范本。

## 6. 插件 API 面

ctx 完整定义：`plugin/src/effect/plugin.ts:35-61`（Context 接口）；宿主实现：`core/src/plugin/host.ts:115-590`（700 行里 475 行就是这个对象字面量）。按域归纳：

```
ctx
├─ app / location / options        元信息 + per-plugin 配置（module.ts:149 注入 operation.options）
├─ storage                         KV 命名空间隔离：key 前缀 plugin:<id 的 UTF-16 hex>:
│                                  （host.ts:622-626——十六进制转义避免 key 注入）
├─ event.subscribe()               Bus 只读流，过滤为 ServerEvent | rpc.*（host.ts:258-268）
├─ 【状态域：transform + reload】
│   agent / command / integration / mcp / model / provider /
│   reference / skill / tool / vcs / websearch / worktree(strategies)
│   └─ transform(editor)：同步编辑器一次性跑；reload() 重放全部 transform
├─ 【hook 域】
│   session.hook / tool.hook / permission.hook / shell.hook / aisdk.hook
│   └─ aisdk.hook 特殊：宿主侧做 providerID 预过滤 + 写回（host.ts:157-188）
├─ 【读 API】
│   agent/model/provider/... .get/.list（跨 Location 时经 LocationServiceMap 重派，host.ts:99-112）
├─ 【动作 API】
│   session.create/prompt/interrupt/wait/...（host.ts:547-589）
│   permission.reply/assert（assert 把拒绝映射为 Tool.Error，host.ts:437-446）
│   generate.text / credential.activate / experimental.terminal.read
└─ rpc                             宿主 RPC client + register（host.ts:119）
```

**rpc.register 的能力面**（§4 图里一行带过，这里说清能干什么）：插件不止消费——还能**定义自己的 RPC 端点并 emit 自定义事件**（`effect/rpc.ts` 的 RpcDomain.register + RpcRegistration.events.emit）。这打开两个方向：**前端可调用插件**（自定义端点进协议面，客户端像调内置 API 一样调它）；**插件间通信**（A 插件 emit 的事件 B 插件经 event.subscribe 收到——rpc.* 前缀正是为此保留的命名空间，:198 行"过滤为 ServerEvent | rpc.*"的后半段）。这也是 §1 说"rpc entrypoint"存在的意义：需要暴露 RPC 的插件走第三入口。

两个版本：Effect 版（`effect/plugin.ts` define，回调返回 Effect）与 Promise 版（`src/README.md`："the async/await equivalent"，setup 可返回 cleanup 函数，`promise/adapter.ts` 桥接）。AGENTS.md 约束：每个 domain 必须扩展 `@opencode/client/effect/api` 的对应 client 接口，只增量添加插件语境函数——保证 ctx API 与官方 client API 形状一致。

storage 域只有 get/set/remove/scan 四个操作（`host.ts:627-648`），scan 自动剥命名空间前缀。

## 7. 生命周期

### 激活时机与就绪门

冷启动时 Location 的插件激活是**异步**的。Session 各入口在读取插件提供的状态前 `yield* Plugin.awaitActivation`（`plugin/service.ts:29-33` 注释：不等待会"admits or fails work the plugins would have shaped"）。实测消费点：`session/prompt.ts:38`、`session/runner/llm.ts`、`session/generate.ts`、`session/skill.ts`、`session/shell.ts`、`session/command.ts`、`filesystem/location-watcher.ts`。

`awaitActivation` 是一个 Latch（`core/src/plugin.ts:27`）：`hold()` 计数>0 时关闭（`plugin.ts:30-38`），supervisor 每次 notify 前先 hold（`supervisor.ts:209-213`），激活流水落定且无新观测请求才释放（`supervisor.ts:236-244`，用 `observed !== target` 判断是否被更新的请求追上）。

### 装载 → 代际 diff

`Plugin.activate`（`core/src/plugin.ts:89-180`）在信号量锁内：
1. 重复 ID 检查（die 于防御，正常情况已被 resolve 层过滤）；
2. **前缀 diff**（`:108-116`）：从左找第一个 (id, revision) 变化的位置，unchanged prefix 的 slot 原地更新定义存活；
3. 变化后缀逆序 close scope（`:129-138`，scope close → 触发插件所有 Registration dispose / promise cleanup）；
4. 新定义依次 `load`（`:139-170`），失败回退旧 revision（§3.3）；
5. inventory 快照 JSON 比对，无变化则不发事件（`:117-122`）——防抖在代际层再做一次。

### 热重载触发流

`supervisor.ts:219-234` 订阅六路变更：config 源变化、**模块文件变化**（§4 的 watch 链）、ManagedPolicy 变化、24h 定时器、`Plugin.Event.Updated`/`SdkPlugins.Updated`、update 状态流。全部汇入 `Queue.sliding(1)`（`:207`，只留最新请求）→ 100ms debounce（`:236`）→ activate。首激活立即执行不 debounce（`Stream.concat(Stream.succeed(0), ...)` 注释，`:235`）。

本地插件改文件 → 指纹变化 → sources.read 重新 import（带新 version 查询参数/cache 失效）→ revision 变化（`module.ts:141` `JSON.stringify([operation, loaded.version])`）→ 代际 diff 装载新版。npm 插件走 `PluginUpdate.check`（24h 缓存，`update.ts:9,39`）标记 outdated，由用户触发 `update(target)`（flock 串行，`update.ts:52-68`）。

## 8. 不变量与设计取舍

从 README/注释提炼（均有出处）：

1. **单通道无特权**：内置 = 内部插件（§5）。收益：宿主只有一条激活路径要测试；代价：内置功能也要遵守代际/序约束（duplicate ID 防御 `supervisor.ts:104`）。
2. **同进程 + 单 effect 副本**（§4 README 条款）：放弃进程隔离换取零 RPC 开销与直接对象引用（event 可变字段链）；用运行时模块拦截把"两份 effect"这个最大风险变成装载期硬错误。
3. **hook 回调无超时、无并发**（`hooks.ts:88-95` 串行 for 循环）：kappa 式简单性——一个坏插件能阻塞但不会静默改变交错语义。错误处理交由失败通道类型系统（Failures=never 除 tool.execute.before）。
4. **失败保留旧代**（`supervisor.ts:85-90`、`plugin.ts:156-166`）：升级失败不等于下线，可用性优先于新鲜度。
5. **常量 revision 的合法性来自不可变性**（`instance.ts:33-37`）：instance 出生列表不变，所以 (id, "instance") 恒等是安全的；同款推理适用于 internal。
6. **transform 失败即禁用插件**（`plugin.ts:50-63`）：状态域被部分写入后无法信任该插件后续 transform，宁可禁用并刷新域，也不带伤运行。
7. **顺序即优先级**：pre → packages → post；instance 绑定插件在 sdk 之后（`supervisor.ts:150-152` 注释："the instance's explicit choices win over globals"）；post（config）天然覆盖一切。
8. **Scope 即卸载语义**：一切注册（hook/transform/rpc）都挂 scope finalizer，close scope = 确定性清理，promise cleanup 也被 await（src/README.md:29-38）。

## 9. 失败与恢复

| 故障 | 策略 | 出处 |
|---|---|---|
| npm 安装/解析失败 | 记 failure + ref，保留旧代运行 | supervisor.ts:59-67, 85-90 |
| 策略 deny | `{blocked}`：禁用且不报错条目 | module.ts:52-59, supervisor.ts:68-74 |
| 入口缺失 | LoadError 显式报错条目 | module.ts:118 |
| default 导出形状不符 | LoadError："must export a default definition with an id and an effect or setup function" | module.ts:125-133 |
| setup 抛错 | warning + slot.error + **回退旧 revision**；旧版也失败才下线 | core/plugin.ts:81-86, 146-169 |
| transform 运行期抛错 | 禁用插件 + 异步刷新受污染状态域（pendingFailures 队列消费者在锁外清理 scope，防死锁："never join them under the activation lock or readiness hold"） | core/plugin.ts:50-63, 182-236, 209 注释 |
| hook 回调抛 defect | 沿触发点传播：当前操作（如 step）失败；除 tool.execute.before 的 Tool.Error 外均为 defect | hooks.ts:23-30；permission assert 映射 host.ts:437-446 |
| 重复 ID | resolve 层转 failure（后出现者），先出现者保留 | supervisor.ts:104-118 |
| activate 整体失败 | logError "failed to reload plugins"，等下一轮触发 | supervisor.ts:239 |
| 清理失败 | 分级 logWarning，不阻断其他插件清理 | core/plugin.ts:210-230 |

超时：**全系统无插件超时**——装载、hook、transform 均无（§8.3 的取舍）。

## 演进注记

旧笔记 4.7（2026-08-03）记的是 `core/src/plugin` + `opencode/src/plugin` 双目录布局。现码（v2.0.30 dev）已是 `packages/plugin` 独立 npm 包：装载器（host/source/runtime）、双 API（effect/promise）、tui/rpc 子入口全部收入 SDK 包；core 侧只剩装配与策略。演进方向清晰：从"宿主内嵌的加载逻辑"变成"可独立发布、带自身 README/PLAN 的插件协议包"，与 9_tool.md 记录的 tool/plugin/*.ts 化（工具以插件形态存在）是同一次架构收敛的两面。旧笔记中"插件进程/worker"类描述已不成立——现行是同进程模型（§4 实证）。

chown donghe:donghe 已确认（ls -la: donghe donghe, 25.5KB）。行号抽查全部来自本轮 sed/read 实测：

```
$ sed -n '23,30p' packages/core/src/plugin/hooks.ts
// Failure channel for each hook event. Only tool execute.before may fail: a Tool.Error rejects the call before it runs.
interface Failures extends Record<keyof Domains, unknown> { ... readonly tool: ToolFailures }
$ sed -n '152,161p' packages/core/src/plugin/supervisor.ts
const pre = [
  ...internal.pre.map((plugin) => ({ ...plugin, revision: "internal", source: { type: "builtin" as const } })),
  ...sdk.all(), ...instance.all(),
]
$ sed -n '276,289p' packages/core/src/plugin/internal.ts
export const list = Effect.fn("PluginInternal.list")(function* () {
  const context = Context.pick(...services)(yield* Effect.context<Requirements>())
  ... effect: (host) => plugin.effect(host).pipe(Effect.provide(context)),
$ sed -n '108,116p' packages/core/src/plugin.ts
const changed = definitions.findIndex((definition, index) => {
  const entry = current[index]
  return entry?.plugin.id !== definition.id || entry.plugin.revision !== definition.revision
})
$ sed -n '30,34p' packages/core/src/session/prompt.ts
  return yield* Effect.gen(function* () {
    yield* Plugin.awaitActivation
    const event = yield* hooks.trigger("session", "prompt", {
$ sed -n '28,34p' packages/plugin/src/runtime.ts
build.onLoad({ filter: createForeignPackageFilter() }, (args) => {
  ... throw new Error(`Cannot load "${target}" from plugin node_modules: ... plugins must use the host's "effect" and "@opencode/plugin" modules.`)
$ sed -n '17,20p' packages/plugin/src/source.node.ts
const hook = registerHooks({
  resolve(specifier, context, nextResolve) {
    if (!context.parentURL || new URL(context.parentURL).searchParams.get("__opencode_reload") !== version)
```

（另覆盖：hooks.ts:88-95 trigger 串行、supervisor.ts:85-90 保留旧代、plugin.ts:156-166 回退、module.ts:52-59 策略门、host.ts:622-626 storage 命名空间、service.ts:29-33 awaitActivation、update.ts:39 24h 缓存、identity.ts:18-32、effect/session.ts:110-124 SessionHooks、effect/tool.ts:24-56、config/plugin/source.ts:115-119。）

## 相关篇

- 前置 [0_effect-infra.md](0_effect-infra.md)：本文大量依赖 Scope/Semaphore/Latch/LayerNode
- [2_config.md](2_config.md)：ConfigPluginSource 的 config 扫描与 Operation 语法
- [8_agent.md](8_agent.md) / [9_tool.md](9_tool.md)：内部插件清单中的 agent/工具注册个体
- [4_session.md](4_session.md)：hook 触发点在 session 流程中的位置（prompt/model-request）
