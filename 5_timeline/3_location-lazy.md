# Location 惰性编译：首请求如何点亮一个项目目录

> 一句话：**`status.ready` 只构建了 25 个全局节点；第一个携带目录的请求才触发 `LocationServiceMap.get(ref)` → `Instance.layer` 编译 53 个 Location 服务（Config/Agent/Tool/Permission/SessionRunner 依赖全在此刻就绪）→ LayerMap 缓存（永不过期）。这就是"服务秒起、首请求稍慢"的原因。**
> 代码基线：v2.0.30，`dev` 分支；行号实测（2026-10-16）。
> [返回 README](README.md) | 上一篇 [2_server-init.md](2_server-init.md) | 下一篇 [4_renderer-boot.md](4_renderer-boot.md)

---

## 0. 全景：两级服务图

```
boot 时（2_server-init.md §4）                首请求时（本篇）
┌─ 全局图：25 节点（进程单例）──────────┐      ┌─ Location 图：53 节点（每目录一份）─────────┐
│ Global Database Bus EventLogger Job  │      │ Location Config Agent Command Reference     │
│ Project Worktree Session(门面)       │◄────│ Provider Model ModelResolver AISDK          │
│ PersistentPty Credential WellKnown   │ 借用 │ Plugin(+Hooks/Supervisor) FileSystem Shell  │
│ LocationServiceMap(未用) …           │ 全局 │ Permission Tool Skill Mcp Form Snapshot    │
└──────────────────────────────────────┘ 节点 │ SessionRunnerLLM/Model Compaction Title …  │
                                           └─────────────────────────────────────────────┘
```

tag 体系（`util/src/effect/app-node.ts:1-13`）：`location: ["global"]`、`global: []`——**Location 可依赖全局，反向禁止**（编译期方向保证）。`Instance.layer` 用 `shared: Node.tags.values.global` 编译，全局节点经父 memoMap 共享（真单例），Location 节点每次构建新实例。

## 0.5 什么是惰性编译（零基础铺垫）

> 来源：读者与 DeepSeek 的概念问答校准后固化。已知 lazy 概念的读者可跳过本节。

**惰性（lazy）= 按需触发，而不是提前全量构建**。对照两种策略：

```
饿汉式（启动即建）：
  服务启动 → 遍历所有可能的目录 → 全部编译好 → 等请求
  问题：目录数量未知、每份编译昂贵（读配置/spawn 插件宿主）、
       用不到的目录纯浪费

惰性编译（用时即建，本篇主角）：
  访问目录 X
    → LayerMap 里有 X 的图吗？
        ├ 有：直接用（缓存命中）
        └ 没有：现在为 X 编译一份 → 放入 LayerMap → 再用
```

两个易错点：

1. **"编译"不是 Java/C++ 那种程序编译**——指解析配置、发现 agent/skill、组装服务依赖、构建图数据结构的**装配过程**（LayerNode 的 compile）。
2. **缓存不是"固定 60 分钟过期"，现码是"永不过期"**（`idleTimeToLive: Duration.infinity`）——早期版本曾是 60 分钟空闲 TTL，后改为 infinity（取舍见 §6"为什么永不过期"）。读到旧资料说 60 分钟的，是过时事实。

代价转移：把编译成本从"每次启动"变成"每个新目录的第一次请求"——这就是"服务秒起、首请求稍慢"的根源。

## 1. 触发点：LocationMiddleware

**`packages/server/src/location.ts`** —— 每个带目录请求的入口流：

```
HTTP 请求（鉴权已过）
│
├─ requestRef(request)（L71-79）── 提取目录
│   ├ ① query 参数 location[directory]
│   ├ ② header x-opencode-directory
│   └ ③ 回退 process.cwd()
│   └► Location.Ref = AbsolutePath.make(directory)
│
└─ layer 中间件（L89-100）
    └ locations.get(requestRef(request))
      └► 把解析出的 Location 上下文提供给内层 handler
```

**含义**：客户端的每次 API 调用都隐式携带目录（生成的客户端自动带 location 参数）；同一目录的后续请求命中 LayerMap 缓存，零编译开销。

## 2. 缓存与规范化：LocationServiceMap

**`packages/core/src/location-service-map.ts`** + **`location-services.ts`** —— 缓存层的解析与生命周期流：

```
LocationServiceMap.get(ref)
│
├─ location-service-map.ts L13-15
│   Service.get(ref) = Layer.unwrap(map => map.get(ref))
│   （把 Map 的 get 解成 Layer 供中间件提供）
│
├─ canonical(ref)（L39-44）★ 缓存键
│   AbsolutePath.make（含 Windows 规范化）+ 保留 workspaceID
│   → /a/b 与 /a/b/ 同键
│
└─ LayerMap.make((ref) => Instance.layer(ref, {replacements}),
                 { idleTimeToLive: Duration.infinity })   ★永不过期
    │
    ├ 命中 ──► 直接返回缓存的整套服务（<1ms）
    │
    └ 未命中 ──► 首次编译 Instance.layer(ref)
        │
        ├ 成功（location-services.ts L28-52）
        │   └ lifecycle.shutdown 存入 build 对象
        │     （进程关停时统一收尾）→ 缓存生效
        │
        └ 失败 ──► 摘除缓存项
            （下次请求重试编译——坏目录不污染缓存）

内层 bindings（L75-83）：
  Instance.node ← 委托本 Map 的 get()（防递归）
  LocationServiceMap.node ← Map 自身
```

## 3. 编译内核：Instance.layer

**`packages/core/src/instance.ts`** —— 一次目录编译的执行流：

```
Instance.layer(ref)
│
├─ L59-112  nodes：53 个 Location 节点（清单见 §4）
├─ L114     graph = LayerNode.group(nodes)
├─ L149     startedAt = performance.now()（启动计时）
│
├─ L152-157 替换链（后写胜前）：
│            默认替换（discovery=false 时 vanilla 配置）
│            → 调用者替换
│            → Location.node.replace(Location.boundNode(ref))
│              InstancePlugins.node.replace(...)
│
├─ L159     ★ LayerNode.compile(graph, {
│              replacements, shared: Node.tags.values.global })
│            ├ 全局节点：借父 memoMap（真单例）
│            └ Location 节点：全新构建
│
├─ L160-171 Layer.catchCause：
│            非 DirectoryNotFoundError 的失败 → die（坏目录快速失败）
│
├─ L172     Layer.tap：注册 LocationLifecycle.shutdown 为 finalizer
│
└─ L173-179 Layer.tap：打日志 "location services booted"
             （directory / workspaceID / ★durationMs——首请求慢多少直接可量）
```

### Location 身份怎么来的

```
Location.boundNode(ref)（core/src/location.ts:19-39）
│
├─ Location.layer(ref)
│   └ Project.Service.resolve(ref.directory)
│     （解析/登记项目——★全局 Project 服务，依赖方向 tag 已保证）
│       └► { directory, workspaceID,
│            project: {id, directory, canonical}, vcs }
│
└─ 包成 makeLocationNode（依赖 Project.node）
```

## 4. 53 个 Location 节点清单（instance.ts L59-112 实测）

按域分组（节点名 → 定义文件）：

| 域 | 节点 |
|---|---|
| **身份与生命周期** | Location（location.ts）、LocationLifecycle（location-lifecycle.ts）、Environment（environment/） |
| **配置与目录内容** | Config（config.ts）、Command（command.ts）、Reference（reference.ts）、Rpc（rpc.ts）、Skill（skill.ts）+ SkillInstructions、InstructionBuiltIns（instructions/builtins.ts）、InstructionDiscovery（instruction-discovery.ts）、ReferenceInstructions |
| **模型与提供者** | Provider（provider.ts）、Model（model.ts）、ModelResolver（model-resolver.ts）、AISDK（aisdk.ts）、Integration（integration.ts）、WebSearch（websearch.ts） |
| **插件** | Plugin（plugin.ts）、PluginHooks（plugin/hooks.ts）、InstancePlugins（plugin/instance.ts）、PluginSupervisor（plugin/supervisor.ts） |
| **文件与终端** | FileSystem（filesystem.ts）、FileSystemSearch（filesystem/search.ts）、LocationWatcher（filesystem/location-watcher.ts）、FileAccess（file-access.ts）、FileMutation（file-mutation.ts）、Shell（shell.ts）、ShellSelect（shell/select.ts）、Pty（pty.ts）、Formatter（formatter.ts）、WorktreeStrategies（worktree/strategies.ts）、Vcs（vcs.ts） |
| **权限与工具** | Permission（permission.ts）、Tool（tool.ts）、ToolOutput（tool-output.ts）、ReadToolFileSystem（tool/read-filesystem.ts）、McpTool（tool/mcp.ts）、Mcp（mcp/index.ts） |
| **会话执行依赖** | SessionInstructions（session/instructions.ts）、InstructionEntry（session/instruction-entry.ts）、SessionRunnerModel（runner/model.ts）、SessionCompaction（compaction.ts）、SessionTitle（title.ts）、SessionContext（context.ts）、SessionRunnerLLM（runner/llm.ts）、Snapshot（snapshot.ts）、Form（form.ts）、Generate（generate.ts）、Image（image.ts） |

**注意**：SessionRunner 的全部执行依赖（LLM 流式、模型解析、压缩、标题、上下文、工具注册表、权限）都在 Location 图里——**drain 一个会话 = 该目录的整套服务已经编译完毕**（对照 [2_server-init.md](2_server-init.md) §4.3：Session 门面在全局图，经未绑定的 `Instance.node` 占位，运行时委托给这里的 Location 实例——`app-node-builder.ts:8-20`）。

## 5. 一次首请求的完整时序

```
客户端 GET /api/agent?location[directory]=/home/me/proj   （鉴权已过）
  → LocationMiddleware（server/location.ts L89-100）
    → requestRef() 提取 /home/me/proj → Location.Ref
    → LocationServiceMap.get(ref)
      → canonical() 规范化（缓存键）
      → LayerMap 命中？ ── 是 ──→ 直接提供上下文，请求继续（<1ms）
            │ 否
            ▼
      → Instance.layer(ref) 编译：
          Location.boundNode → Project.resolve（登记项目）
          LayerNode.compile(53 节点, shared: global)
            ├─ 借用全局：Database/Bus/Job/…
            └─ 新建：Config（读 .opencode/ 层叠）、Agent/Tool 注册表、
               Permission、ModelResolver、PluginSupervisor（spawn 插件宿主）…
          Layer.tap：finalizer + "location services booted" 日志（durationMs）
      → 缓存进 LayerMap（Duration.infinity）
  → 内层 handler（AgentHandler 等）在 Location 上下文中执行
  → 响应

下一个同目录请求：缓存命中，零编译
服务重启：缓存全失（内存态），首个请求重新编译
```

## 6. 设计取舍

- **为什么惰性**：服务进程可能面对任意多个项目目录；启动时全编译不可行（每个目录要读配置、spawn 插件宿主）。惰性 + 按需 = 启动 O(1)、成本发生在使用处
- **为什么永不过期**：过期意味着目录服务随时可能重建（插件重 spawn、watcher 重建），中断用户工作流；宁可用内存换稳定。（早期版本是 60 分钟 TTL，现已改为 infinity）
- **为什么失败摘除**：坏目录（权限/不存在）不该污染缓存——失败即摘，环境修复后自动重试
- **为什么 shared: global**：Database/Bus 这类进程级服务若被 Location 图复制，事件溯源就分裂了——共享是正确性要求，不只是优化
- 红线出处：根 `AGENTS.md`「Session Core」（Location 作用域约束）、`specs/v2/session.md`（执行是 Location 作用域）

## 相关篇

- [2_server-init.md](2_server-init.md)——全局 25 节点怎么来的
- [../1_architecture/5_patterns.md](../1_architecture/5_patterns.md)——LayerNode/LayerMap/同名目录惯例
- [../2_tech-stack/2.2_server/3.4_effect-guide.md](../2_tech-stack/2.2_server/3.4_effect-guide.md) §4——Layer/Service 概念
- [5_prompt-chain.md](5_prompt-chain.md)——编译完成后 drain 怎么跑
