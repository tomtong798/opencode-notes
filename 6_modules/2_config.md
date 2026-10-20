# 配置子系统详解（发现 → 归一化 → Entry 流 → 配置驱动插件）

> 一句话：**opencode 的配置不是一个"合并成一个大对象"的静态文件，而是一条"多层发现 → 逐文档归一化（V1/V2 双形状收敛）→ 有序 Entry 列表 + 快照读取"的流水线，再由 18 个配置插件把 Entry 流翻译成 agent/provider/mcp/policy 等注册表变更——配置是插件生态的输入，而不是终点。**
> 代码基线：v2.0.30，dev 分支（e25b7abf89）；行号实测（2026-10-19）。
> 权威出处：`packages/schema/src/config.ts` 及 `packages/schema/src/config/*`（schema 即权威——`decodeOptions = { errors: "all", onExcessProperty: "ignore" }`，多余字段静默忽略）。
> [返回 README](README.md) | 前置 [0_effect-infra.md](0_effect-infra.md)（Config.node 是 location 级 LayerNode） | 关联 [8_agent.md](8_agent.md)（agent 配置消费面）/ [6_context.md](6_context.md)（instruction 发现）

---

## 0. 模块地图

三个包、四块代码：

```
packages/schema/src/
├─ config.ts                ★ Config.Info 顶层 schema + Document/Directory/Entry + config.updated 事件（142 行）
└─ config/（18 文件，全是叶子 schema）
   ├─ agent.ts      (22)     model.ts (37)      provider.ts (105)   mcp.ts (22)
   ├─ command.ts    (14)     policy.ts (13)     plugin.ts (14)      reference.ts (23)
   ├─ compaction.ts (14)     experimental.ts(17) watcher.ts (8)     websearch.ts (17)
   ├─ formatter.ts  (13)     lsp.ts (19)        media.ts (15)       tool-output.ts (9)
   └─ warming.ts    (18)     worktree.ts (10)   shell.ts (10)

packages/core/src/
├─ config.ts                ★ 门面：Config.Service/layer/latest()（437 行）
├─ config/
│  ├─ discovery.ts  (145)   目录/文件发现 + 系统目录/MDM plist 定位
│  ├─ normalize.ts  (843)   ★ V1/V2 双形状 → 规范 V2 形状（诊断驱动）
│  ├─ variable.ts   (80)    {env:VAR} / {file:path} 文本替换
│  ├─ watch.ts      (41)    watcher 订阅计划（目录 + 父目录兜底）
│  ├─ markdown.ts   (40)    gray-matter 容错解析（frontmatter）
│  └─ plugin/（21 文件）    配置驱动的插件源（§5 详述）
│     ├─ source.ts (197)    plugins 数组 + {plugin,plugins}/ 目录 → Operation 流
│     ├─ entry-observer.ts(25) config.updated 订阅 + entries 快照（多数插件共用）
│     ├─ agent.ts (212)  provider.ts (180)  skill.ts (193)  command.ts (250)
│     ├─ instruction.ts(239) reference.ts (67) policy.ts (65) compatibility.ts (77)
│     └─ compaction/formatter/image/lsp… mcp.ts (57) shell/snapshot/websearch/…
└─ v1/config/（7 文件，V1 遗产）
   ├─ migrate.ts   (263)    ★ V1 → V2 字段映射（agent/command/mcp/provider/model）
   ├─ provider.ts  (122) / provider-options.ts (58) / agent.ts (69) / mcp.ts (63)
   └─ permission.ts (42) / command.ts (13) / plugin.ts (9) / error.ts (18)
```

角色一句话：**schema 定义形状，discovery 找文件，normalize 收敛形状，config.ts 门面持有快照并广播变更，config/plugin/* 把配置灌进各领域注册表，v1/config 只剩"旧形状的解码器 + 迁移函数"**。AGENTS.md 风格约束：config 模块顶部统一 `export * as ConfigXxx from "./xxx.js"` 自导出。

## 1. 心智模型

**没有"合并后的大配置对象"。** 每个配置文件解析成一个 `Document`（`schema/config.ts:121-125`：`{type:"document", path?, info}`），每个配置目录产生一个 `Directory` 占位（`:127-130`），全部按**优先级从低到高**排成 `Entry[]`。优先级仲裁推迟到消费侧：标量字段用 `Config.latest()`（`core/config.ts:24-27`，`findLast` 找最后一个定义该字段的 document——数组末尾即最高优先级）；map 型字段（agents/mcp servers/commands）由各插件按序遍历全部 document 逐个 upsert。

加载优先级（`core/config.ts:281-290` 实测拼接顺序，越靠后优先级越高）：

```
低 ──→ 高
managedPolicies          # 托管策略被拆出、反转后垫底（保住"文件权威"，:250-266）
wellknown                # 集成清单（远程拉取，失败回退缓存副本 :156-165）
globalSupplementary      # ~/.config/opencode/opencode.json{,c}（:273）
explicit                 # --file 指定（Options.file，:226-231）
direct                   # 项目树里散落的 opencode.json{,c}（祖先回走，近者胜，:124 toReversed）
projectSupplementary     # 各祖先 .opencode/ 目录（:274-280）
content                  # OPENCODE_CONFIG_CONTENT 环境变量内联配置（:232-244）★环境变量的位置
managedSettings          # 系统目录(/etc/opencode 等) + macOS MDM plist，压过一切（:246-253）
```

注意：**环境变量不是一层配置**——它只通过 `{env:VAR}` 文本替换（§3）和 `OPENCODE_CONFIG_CONTENT` 内联内容参与；分层是纯文件拓扑。项目发现是 `fs.up()` 从 cwd 向上走到底（`util/fs-util.ts:165`），每层探测 `.claude/.agents/.opencode/opencode.json(c)`（`discovery.ts:85`），全局根会被剔除防止"回灌"（`discovery.ts:102-107` 注释："Global sources must not re-enter through the project walk"）。

```
            文件系统                     Config.Service（每 Location 一份）
 ┌──────────────────────────┐
 │ /etc/opencode/*.json  ───┼─┐
 │ MDM plist (darwin)    ───┼─┤ managed（settings 部分最高优先）
 │ ~/.config/opencode/   ───┼─┤
 │  ~/.claude ~/.agents  ──┼─┤ compatibility roots（只喂 compatibility 插件）
 │ cwd 祖先每层:          │  │
 │   opencode.json(c)    ──┼─┤ direct
 │   .opencode/ .claude/ ──┼─┘
 └──────────────────────────┘
        │ Watcher（watch.ts plan：目录 + 父目录兜底 :33-36）
        ▼
  fs update → PubSub(reloads, sliding 1) → debounce 100ms（config.ts:336-342）
        ▼
  reload()（Semaphore 串行，:318-332）
    discover → load → normalize → 深比较（isDeepStrictEqual）
    无变化 = 静默；有变化 = 替换 configs + Bus 发布 config.updated
        ▼
  两类消费者：① ConfigEntryObserver 型插件（快照刷新）
             ② config.changes() 原始 fs 流（agent/command/instruction 自行过滤+debounce）
```

## 2. 配置 Schema 速览

`Config.Info`（`schema/config.ts:26-114`）顶层字段（全部 optional，蛇形命名）：

| 字段 | schema | 说明 |
|---|---|---|
| `model` | `ConfigModel.Selection` | `"provider/model#variant"` 短串或显式结构（model.ts:15-25 Union+transform） |
| `default_agent` | String | 默认主 agent |
| `shell` / `username` | String | 默认 shell / 展示名 |
| `update` | `"disable"|"notify"|"auto"` | 升级策略（`:39-41`） |
| `share` / `enterprise` | Literals / Struct | 分享策略 / 企业服务 URL |
| `permissions` | `Permission.Ruleset` | 有序工具权限规则（最后匹配胜） |
| `policies` | `ConfigPolicy.Info[]` | `provider.use / tool.use / integration.use` 三类 allow/deny（policy.ts:6-11） |
| `agents` | `Record<String, ConfigAgent.Info>` | agent 覆写/自定义（形状详见 8_agent §配置侧） |
| `commands` / `instructions` / `skills` | Record / String[] / String[] | 命令、指令路径、skill 路径 |
| `mcp` | `ConfigMCP.Info`（`timeout` + `servers`，mcp.ts:20-22） | MCP 服务器与三段超时 |
| `providers` | `Record<String, ConfigProvider.Info>`（105 行） | 自定义 provider/models |
| `plugins` | `(String | {package, options})[]`（plugin.ts:14-16） | 插件启停指令 |
| `references` / `websearch` / `worktree` / `warming` | 各叶子 schema | 外部上下文 / 搜索选择 / worktree / 保活 |
| `media` / `tool_output` / `compaction` | 叶子 schema | 媒体 / 截断 / 压缩 |
| `watcher` / `formatter` / `lsp` / `snapshots` | 叶子 schema | 观察忽略 / 格式化 / LSP / 快照开关 |
| `experimental` | `ConfigExperimental.Info` | §7 |
| `$schema` | String | JSON schema 引用 |

写入 API 极小化：`Config.Patch` 目前只允许改 `shell`（`schema/config.ts:116-118`）。

## 3. 加载与合并管线

单个文件 → Document 的五步（`core/config.ts:142-149` loadFile）：

1. **读取**：`fs.readFileStringSafe`，读不到返回 undefined（该文件静默缺席）。
2. **变量替换**（`config/variable.ts:26-33`）：`{env:VAR}` 用 `input.env ?? process.env` 替换，缺失置空串；`{file:path}`（`:35-79`）支持 `~/` 展开与相对配置目录解析，行首 `//` 注释里的 token 不替换（`:49-53`），文件缺失只告警并替换为空——注释（`:66-67`）："a missing {env:VAR}…so one stale secret reference cannot take down every Location"。**旧笔记提的变量替换现码仍在，且升级为 wellknown 场景注入凭据的通道**（`config.ts:151-154`：manifest auth env + credential key 组成替换变量集）。
3. **JSONC 解析**：`jsonc-parser` `parse(..., {allowTrailingComma: true})`（`config.ts:112`），解析错误 → 整文档丢弃（§9）。
4. **归一化**：`ConfigNormalize.normalize(input)`（§6 前奏）——V1 形状字段就地迁移为 V2，产出 diagnostics。
5. **Schema 校验**：`decodeInfo(result.encoded)`（`:108-109`），失败 → 整文档丢弃。

**合并算法的真相**：normalize 只做**文档内**的 legacy/native 字段合并——`mergeMaps`（normalize.ts:763-776）是"浅层按键覆盖 + 冲突诊断"（`conflict` 诊断："retained native value over legacy value"），`prefer`（`:804-810`）是"native 优先 + 冲突诊断"。**没有跨文档深合并、没有数组合并**——跨文档优先级全靠 Entry 排序 + `latest()`/按序 upsert。数组型字段（plugins/policies/permissions/instructions）在各文档独立保留，消费侧拼接（如 plugins：normalize.ts:186-189 legacy 在前 native 在后；policy 插件按"authored 反转 + Console 最后"重排，policy.ts:23-24）。

目录加载 `loadDirectory`（`config.ts:211-218`）按 `names = ["opencode.json", "opencode.jsonc"]`（discovery.ts:12）顺序尝试，**每个存在的都成为一个 Document**（同目录两份并存，靠 latest 的 findLast 自然让 jsonc 序号更高——但请注意 names 顺序 json 在前）。macOS MDM plist 用 `plutil -convert json` 外部进程转 JSON（`:192-196`，10s 超时），并剥离 Payload* 元数据键（discovery.ts:42-57）。

## 4. Config.Service 与热重载

`Config.Service`（`core/config.ts:60`）接口四件套（`:29-45`）：`entries()`（返回当前快照，低→高）、`compatibility()`（`~/.claude`、`~/.agents` + 项目内同名目录，喂 compatibility 插件）、`changes()`（**原始 fs update 流**——注释明说："Config owns root topology…domain owners filter this feed"，`config/plugin/agent.ts` 等据此自行 debounce）、`update(patch)`（JSONC 保格式写入：`jsonc-parser` `modify/applyEdits` 只动 `shell` 键再落盘，`config.ts:389-399`）。

热重载三要素（均实测行号）：

- **watch 计划**：`ConfigWatch.plan`（watch.ts:10-41）对 system/global/project 根做目录订阅（忽略 `node_modules`/`.git`），对根外散文件（direct/claude/agents/explicit）按父目录分组做 `entries` 型订阅——`:33-36` 注释："Keep a parent watch for each root so deletion/recreation is observable"（目录删除后重建仍能被看见）。watch 集合随每次 discover 增量 reconcile（`config.ts:301-316`，FiberMap 按 key 收敛，`onlyIfMissing` 幂等）。
- **防抖与串行**：fs 事件 → `reloads` PubSub（sliding 1，天然合并突发）→ `Stream.debounce("100 millis")`（`config.ts:336-342`）→ `reload()`。reload 全程持 `reloadLock` 信号量（`:331`）；**先订阅再初始加载**（`:334-335` 注释："Subscribe eagerly so synchronous watch readiness isn't dropped"），且 readiness 重扫本身也会再触发一次 reload（`:298-299` 注释："Readiness rescans recover writes made before a watch attached"）。
- **变更判定**：新旧 configs `isDeepStrictEqual` 深比较，且 compatibility 根变化单独判等（`:323-326`）——**无变化不发事件**。变化则 `Bus.publish(Event.Updated, {})`（`:329`，即 `config.updated`，schema/config.ts:137-140 定义）。

除 fs 外还有三个 reload 触发源：凭据切换（仅当命中某 wellknown 集成，`:343-354`）、wellknown 清单更新（`:355-360`）、每 10 分钟的 wellknown 定时刷新（`:361-379`）。

`config.updated` 如何触发插件重装：`PluginSupervisor` 直接订阅它作为 activate 触发器之一（`plugin/supervisor.ts:226`：`watch(bus.subscribe([Event.Updated, SdkPlugins.Updated]))`）→ 经 sliding queue + 100ms debounce（`:245-247`）→ `activate()` 重算全部插件代次。**配置插件自己不被"重装"，而是内部刷新**：`ConfigEntryObserver.observe`（`config/plugin/entry-observer.ts:12-27`）订阅 `config.updated` → 重读 `config.entries()` 快照 → 调领域 `reload()`；`:23-25` 注释专门处理首读与订阅之间的竞态（"Close the race between the first read and establishing the subscription"）。

## 5. 配置驱动的插件源（核心）

`internal.ts` 把 16 个 Config*Plugin 全部排在 `post` 组（`plugin/internal.ts:252-268`）——**先跑工具类内置插件，配置插件最后跑，后写者胜**，所以配置能覆写内置默认。两个例外机制：`guarded` 集合（`:270-272`）保护 `opencode.config.policy` 不被 `plugins: ["-…"]` 移除（注释："Repository config must not switch off policy enforcement…"）；`ConfigPluginSource`（source.ts）不是插件而是插件**供给层**，由 supervisor 消费（supervisor.ts:130、162）。

按消费模式分三类：

**A. EntryObserver 型（标量/map 一次性灌入，8 个）**——`observe + transform` 两步，配 `latest()` 取值：
- `shell`（shell.ts:15-20）：`Config.latest(entries,"shell")` → `shell.transform.configure`。
- `snapshot` / `tool-output` / `websearch` / `worktree` / `image`：同构，各自 latest 一个字段 configure。
- `compaction`（compaction.ts:15-24）：**不取 latest 而按序遍历全部 document**，逐个 `editor.configure({auto,buffer,keep})`——后写的文档覆盖前写的键，等价于分层合并。
- `location-watcher`（location-watcher.ts:14-21）：聚合全部 `watcher.ignore` 数组。

**B. 自管 watcher 型（文件+配置双源，4 个）**——不走 entry-observer，订阅 `config.changes()` 原始 fs 流 + `isXxxSource` 路径过滤 + 自建 100ms debounce：
- `agent`（agent.ts:44-93，机制详见 8_agent §配置来源）：配置文件 `agents:` 键 + `{agent,agents,mode,modes}/**/*.md` frontmatter（经 `config/markdown.ts` 容错解析）→ `ctx.agent.transform` upsert。`:47-48` 注释解释单触发流设计："One trigger feed serializes reloads and shares one debounce window"。
- `command`（command.ts:44-70）：`commands:` 键 + 目录 `.md` → `ctx.command.reload()`。
- `instruction`（instruction.ts:21+，发现细节归 6_context §3）：`instructions:` 路径/URL + AGENTS.md 祖先链。
- `compatibility`（compatibility.ts:18-77）：**唯一消费 `config.compatibility()` 的插件**——把 `~/.claude`、`~/.agents` 及项目同名目录下的 skills 翻译成 skill 注册表项。

**C. 注册表翻译型（map/规则语义，4 个）**：
- `provider`（provider.ts:17-101）：`providers:` 键 → integration method（key/env）+ 自定义 models/variants/cost；`sources.defaultModel = Config.latest(loaded.entries,"model")`（`:101`）。Azure/Vertex 旧 provider 走 migrate 的专用分支（§6）。
- `mcp`（mcp.ts:33-52）：全局 `timeout` 按文档序 `Object.assign` 浅合并，servers 同名后写胜（`servers.set`），每 server 超时 `{...timeout, ...server.timeout}` server 级覆盖全局——这是全库最接近"深合并"的地方。
- `policy`（policy.ts:17-59）：`policies:` + ManagedPolicy（Console 组织语句）。authored 文档**反转**后 Console 语句殿后（`:23-24` 注释："user-global policy outranks repository policy; organization statements…have the final say"）；对 provider/mcp/skill 注册表做 deny 即 remove，并挂 `permission.hook("evaluate")` 拦截 `tool.use` deny。
- `skill`（skill.ts:193 行）+ `skill-file.ts`：`skills:` 路径 → skill 注册表。

`ConfigPluginSource`（source.ts）单独说：`operations()` 把 `plugins:` 数组解析成 `{type:"add",target,options,mtime?}` / `{type:"remove",target}`（parse，`:100-112`；`"-name"` 前缀即 remove），叠加配置目录下 `{plugin,plugins}/` 自动发现（`PluginSourceDirectory.names`，source-directory.ts:7）。相对路径相对**配置文件所在目录**解析（`:135-138`），file:// URL 支持；mtime 取入口文件 + package.json 的最大值供增量热载。`:87-88` 注释："Explicit config is applied last so it can remove auto-discovered packages"——显式 remove 能干掉自动发现的插件。配置根外的插件目标由它自己补 watch（`:43-52` 注释："…a stale watch after a config edit costs one deduped fs handle"）。

## 6. V1 迁移

V1 已无加载器，只剩"**形状解码器 + 迁移函数**"，全部由 `normalize.ts` import（normalize.ts:28-34）。每个文档先按 V1 字段尝试解码，成功即走 `ConfigMigrateV1.migrateXxx` 转成 V2 形状，再与 native V2 字段 `mergeMaps`/`prefer` 合并。典型字段走查（`v1/config/migrate.ts` 实测）：

**① permission → permissions + 动作改名**（migrate.ts:17-32 + normalize.ts:183-186）：V1 的 `permission: {bash: "ask", edit: [{...}]}`（`v1/config/permission.ts:33-41`，对象按键序保序解码——`:6-8` 注释强调"last-match-wins, so the order an author writes keys in is policy"）翻成 `Permission.Rule[]`；工具名经 `normalizeAction`（migrate.ts:28-33）：`write|patch→edit`、`task→subagent`、`bash→shell`——旧规则在 V2 工具名下继续生效。

**② agent 单体形状拆解**（migrate.ts:35-64）：V1 agent 的顶层 `options`（temperature/top_p 等 provider 参数）被收进 `request.body`，`prompt→system`、`disable→disabled`、`maxSteps→steps`；`model` 字符串经 `modelSelection`（`:69-78`）用正则 `/^[^/#]+\/[^#]+$/` 拆成 `{providerID, model, variant?}`，非 `prov/model` 形状直接弃用。`small_model` 在 normalize 侧被翻译成 `agents.title` 的 model（normalize.ts:134-147）。

**③ MCP 单一超时 → 三段超时**（migrate.ts:79-83）：V1 一个 number 管所有阶段，V2 是 `{startup, catalog, execution}`——裸数直接复制三份；normalize 还接受 native server 里的裸数超时（normalize.ts:309-318 `expandServerTimeouts`，注释："…so one V1-shaped field does not drop the whole server"）。另两个易漏点：`enabled` 布尔翻转为 `disabled`（migrate.ts:84）；provider 旧 ID 改名 `azure-cognitive-services→azure`、`google-vertex-anthropic→google-vertex`（`:177-181`），且顶层 `model` 的 providerID 也过一遍改名（normalize.ts:203）。

normalize 自身还处理一批"不必走 V1 schema"的浅改名：`autoupdate→update`（true/false/"notify" → "auto"/"disable"/"notify"，normalize.ts:73-88）、`autoshare→share`、`snapshot→snapshots`、`attachment→media`、`reference→references`、`command→commands`、`provider→providers`、`tools→permissions`、`plugin→plugins`（`:186-188` 元组 `[pkg, options]` → `{package, options}`）、`enabled_providers/disabled_providers` + `experimental.policies` → 生成式 `policies` 规则（`:414-444`，deny-all + 逐个 allow）。

## 7. Experimental 开关

现码**只剩两个真开关 + 一个废弃位**（`schema/config/experimental.ts:6-16`）：

| 键 | 类型 | 说明 |
|---|---|---|
| `portable_shell_scanner` | Boolean（默认 false） | 实验性便携 shell 权限扫描器 |
| `subagent_depth` | NonNegativeInt（默认 1） | 子 agent 最大嵌套深度 |
| `policies` | ConfigPolicy.Info[] | **已废弃**——归并进顶层 `policies`（normalize.ts:414-447） |

normalize 会显式剔除一批已退役的实验键并打 `unsupported` 诊断（normalize.ts:46-52）：`disable_paste_summary / batch_tool / openTelemetry / primary_tools / continue_loop_on_deny`；顶层同样剔除 `logLevel / server / subagent_depth（顶层旧位）/ layout`（`:45`）。注意顶层 `subagent_depth` 是 unsupported 而实验内的同名校验保留——旧笔记若在顶层见过它，现码已迁移。

## 8. 不变量与设计取舍

从注释与结构提取（均为源码注释直译/意译）：

1. **文件权威（file-based authority）**：托管设置最后加载、优先级最高，但托管 **policies** 被拆出反转垫底——组织能锁死策略结果却不能替用户写策略（config.ts:246-249）。
2. **全局不回灌**：项目祖先回走必须排除全局根/全局文件，防止 `~/.config/opencode` 被当项目配置重复计入（discovery.ts:101-107）。
3. **无变化不广播**：深比较相等（含 compatibility 根）即 return，下游零抖动（config.ts:326）。
4. **Config 只管拓扑，语义归域主**：`changes()` 吐原始 fs 事件，"domain owners filter this feed for the source files they parse and rebuild their own state"（config.ts:37-42）——这是 agent/command/instruction 各自 debounce 的架构依据。
5. **先订阅后首读**：watcher readiness、entry-observer、agent 插件三处都有同款竞态注释（config.ts:334 / entry-observer.ts:24 / agent.ts:46-49）。
6. **stale 引用不致命**：`{env:}`/`{file:}` 缺失替换为空（variable.ts:66-67）、wellknown 刷新失败保留缓存副本（config.ts:156-157）——单点坏引用不拖垮整个 Location。
7. **guard 不可关**：policy 插件与 Console 连接不可被仓库配置 remove（internal.ts:270-271）。
8. **schema 是浏览器安全的合同**（schema/AGENTS.md）："Schema values should be serializable contract definitions, not service implementations"——Config.Info 全 optional + `optional()` helper 确保 undefined 键不编码。

## 9. 失败与恢复

降级行为按粒度递减（全部实测）：

- **JSONC 语法错误**：jsonc-parser 报错 → 整文档丢弃 + warning 日志"rejected malformed JSON or JSONC document"（config.ts:113-120），其余文档照常。
- **根不是对象**：normalize 直接 `rejected`（normalize.ts:58-63）。
- **字段校验失败**：`invalid` 诊断 → **只跳过该字段**（"skipped malformed recognized recognized value"，normalize.ts:825-827），文档其余部分保留；全部诊断只 logWarning 不抛异常（config.ts:123-130）。最终 decodeInfo 失败才整文档丢弃（`:134-139`）。
- **不认识的字段**：`onExcessProperty: "ignore"` 静默忽略（`:108`）——向前兼容的关键。
- **已退役字段**：`unsupported` 诊断 + 剔除，不报错（normalize.ts:819-822）。
- **legacy/native 冲突**：`conflict` 诊断，native 胜（`:810-815`）。
- **wellknown/MDM plist 失败**：plist 转换超时 → 空数组 + warning（config.ts:197-199）；wellknown 拉取失败 → 缓存副本兜底（`:160-165`）。
- **reload 失败**：`catchCause` 记日志，旧 configs 原样保留（`:339`）；watch fiber 失败同样 catchCause 存活（`:311-314` 之外的 supervisor 同款）。
- **`{file:}` 指向不可读文件**：warning + 空串替换（variable.ts:68-73）。
- **写回失败**：`update()` 的 JSONC 编辑异常包装成 `FSUtil.FileSystemError` 抛出（config.ts:396-398）——写路径不降级，读路径全降级。

## 演进注记

旧笔记 4.1（2026-08-03 基线）记录的是 `core/src/config` + `opencode/src/config` **双轨加载**：V1 运行时自带一套独立的配置发现/合并，V2 另起一套，两轨并存。现码（v2.0.30）：`opencode/src` 已整体删除，V1 只剩 `core/src/v1/config/` 的 7 个 schema/迁移文件——**单轨加载 + 文档级迁移**：所有文件（无论写的是 V1 还是 V2 形状）都走同一条 discovery→normalize→decode 管线，V1 形状在 normalize 内被就地翻译，加载器不再有第二套。`1_architecture/6_v1-to-v2-evolution.md` 已把"双形状配置"标记为**活跃的发布特性**（用户旧配置无需改写即可升级）。另一处演进：旧笔记时代的"变量替换"只是文本便利，现码把它升级为 wellknown 凭据注入通道（config.ts:151-154）；`share/update` 等布尔字段也从自由布尔收敛为 `Schema.Literals` 闭集。

## 相关篇

- [0_effect-infra.md](0_effect-infra.md)：Config.node 的 location 作用域与 LayerNode 组装
- [8_agent.md](8_agent.md)：§配置来源——config/plugin/agent.ts 的 upsert/权限 push 语义（本篇只写机制框架）
- [6_context.md](6_context.md)：§3——config/plugin/instruction.ts 的 AGENTS.md 发现与三路来源（本篇不重复）
- [11_permission.md](11_permission.md)：Permission.Ruleset 与 policy 插件的 evaluate hook 交叉
- [../1_architecture/6_v1-to-v2-evolution.md](../1_architecture/6_v1-to-v2-evolution.md)：V1 遗产全景（本篇只覆盖 config 切面）
