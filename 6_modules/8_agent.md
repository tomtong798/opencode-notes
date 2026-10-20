# Agent 子系统详解（空注册表 + 插件注入的行为包：从注册到会话执行）

> 一句话：**Agent 是 opencode 的"行为包"抽象——一个 ID 键控的空注册表（`Agent.Service`），所有 agent 定义（含内置 7 个）都由插件在启动时通过 `ctx.agent.transform(editor)` 注入；每个 Agent 是 prompt + 权限规则集 + 模型偏好 + step 上限 + subagent 能力的组合，会话只持有 agent ID，执行时才解析为完整定义。**
> 代码基线：v2.0.30，dev 分支（commit e25b7abf89）；行号实测（2026-10-19）。
> 权威出处：仓库 AGENTS.md（无 spec 对应篇）+ 实测源码。
> [返回 README](README.md) | 前置 [4_session.md](4_session.md) | 关联 [6_context.md](6_context.md)（agent system prompt 进入上下文）、[9_tool.md](9_tool.md)（权限规则收窄工具快照）

---

## 0. 模块地图

```
packages/
├─ schema/src/
│   ├─ agent.ts            ★ Agent.Info schema + Event.Updated（59 行）
│   ├─ config/agent.ts     Config.Agent（用户配置侧形状，22 行）
│   └─ permission.ts       Permission.Ruleset = Array(Rule)（L66）
├─ core/src/
│   ├─ agent.ts            ★ 空注册表 + State.Transformable 服务（116 行）
│   ├─ plugin/agent.ts     ★ AgentPlugin：内置 6 agent 注册（143 行）
│   ├─ plugin/plan.ts      ★ PlanPlugin：第 7 个内置 agent "plan" + 模式提醒（120 行）
│   ├─ plugin/internal.ts  内置插件清单（AgentPlugin L223 / PlanPlugin L224 / SubagentTool L245 / ConfigAgentPlugin L255）
│   ├─ config/plugin/agent.ts ★ 配置文件 → agent 定义（.md frontmatter / V1 迁移，212 行）
│   ├─ session/
│   │   ├─ context.ts      ★ select(): agent 解析 + AgentNotFoundError（208 行）
│   │   ├─ session.ts      switchAgent → AgentSelected durable 事件（426 行）
│   │   ├─ projector.ts    AgentSelected → session 表 agent 列更新（L555-563）
│   │   ├─ message-updater.ts "session.agent.selected" → agent-switched 消息（L91-100）
│   │   ├─ model-request.ts baseTranscript: agent.system → system prompt（L100-118）
│   │   └─ runner/
│   │       ├─ llm.ts      step 计数与 agent.info.steps 上限（L226）
│   │       ├─ step.ts     toolChoice=none 时拒绝工具（L92-94）
│   │       └─ max-steps.ts MAX_STEPS_PROMPT 注入文本（16 行）
│   ├─ tool/plugin/subagent.ts ★ subagent 工具：agent 定义派生子会话（311 行）
│   └─ managed-policy.ts   Console 组织策略（与 agent 无直接耦合，75 行）
├─ plugin/src/effect/agent.ts  AgentEditor/AgentDomain 插件 API 形状（17 行）
└─ server/src/handlers/agent.ts  HTTP: agent.list / agent.get（27 行）
```

managed-policy.ts 实测**与 agent 权限无直接关系**——它承载 OpenCode Console 下发的组织级策略语句（`statements()` L50-64），汇入的是**配置策略**评估（config policy），不触碰 agent.permissions。故本篇只在 §7 提一句边界。

---

## 1. 心智模型

**Agent = 行为包（behavior bundle）**。它不是执行实体——执行实体是 Session+Runner；Agent 只是五样东西的命名打包：

1. **system prompt**（`system` 字段；缺省回退到全局 harness prompt）
2. **权限规则集**（`permissions: Permission.Ruleset`，最长的字段）
3. **模型偏好**（`model: Model.Ref?` + `request` provider 覆盖）
4. **step 上限**（`steps?: PositiveInt`）
5. **subagent 能力**（`mode: "subagent" | "primary" | "all"` 决定能否被 subagent 工具派生）

```
            注册时（Location 启动）                    执行时（每次 Step 前）
┌──────────────────────────────┐      ┌──────────────────────────────────────┐
│ AgentPlugin (plugin/agent.ts)│      │  SessionContext.select()             │
│ PlanPlugin    (plugin/plan)  │      │    agents.select(session.agent)      │
│ ConfigAgentPlugin (配置文件)  │      │         │                           │
│ 用户插件 (ctx.agent.transform)│      │         ▼                           │
│         │ editor.update(id)  │      │  Agent.Selection {id, info}          │
│         ▼                    │      │         │                           │
│  Agent.Service 注册表 (Map)   │─────▶│  merge(agent.permissions,           │
│  {build, plan, general,      │      │       session.permissions)          │
│   explore, compaction,       │      │         ▼                           │
│   title, summary, 用户自定义} │      │  Tool.snapshot(permissions) 工具快照 │
└──────────────────────────────┘      │         ▼                           │
   会话只存 agent ID（session 表       │  baseTranscript: agent.system 或     │
   agent 列）；切换=发 AgentSelected   │  SystemPrompt.make(tools) → system   │
   durable 事件                        └──────────────────────────────────────┘
```

关键解耦：**会话的持久化状态只有一个 `Agent.ID` 字符串**（session 表 agent 列，projector.ts:460/560），agent 定义本身是易变的运行时数据（配置文件改动会热重载）。同样的历史，换一个 agent 就得到完全不同的工具集、权限和 system prompt——这就是"行为包"的含义。

---

## 2. Agent 定义结构

`packages/schema/src/agent.ts:23-35`，逐字段：

```ts
export const Info = Schema.Struct({
  id: ID,                                    // branded string "Agent.ID"
  name: Name,                                // branded string，展示名（如 "Build"）
  model: Model.Ref.pipe(optional),           // {providerID, id, variant?} 模型偏好
  request: Provider.Request,                 // provider 请求覆盖 {settings, headers, body}
  system: Schema.String.pipe(optional),      // 自定义 system prompt（缺省用 harness prompt）
  description: Schema.String.pipe(optional), // 展示/路由描述（subagent 工具会拼进工具描述）
  mode: Schema.Literals(["subagent", "primary", "all"]),
  hidden: Schema.Boolean,                    // 隐藏则不出现在 UI 可选列表
  color: Color.pipe(optional),               // UI 展示色
  steps: PositiveInt.pipe(optional),         // step 上限（见 §5）
  permissions: Permission.Ruleset,           // Permission.Rule[]（action/resource/effect）
})
```

字段语义要点：

- **`request`** 是必填但默认空的 `{settings:{}, headers:{}, body:{}}`（schema/agent.ts:43）——配置侧可以对单个 agent 追加 temperature/top_p 等 provider 请求参数（config/plugin/agent.ts:113-116 `Object.assign` 合并 headers/body）。
- **`permissions` 是数组、有序**——评估时 `findLast` 取最后匹配（managed-policy.ts:72 同款语义，permission.ts 内 `evaluate` 亦按序）。所以"先 push 全量 deny 再 push 白名单 allow"是合法写法（explore 正是这么做的，见 §3）。
- **`mode` 三态**：`primary` = 只能作为会话主 agent；`subagent` = 只能被 subagent 工具派生；`all` = 两者皆可。subagent 工具在 L148-149 拒绝 `mode === "primary"` 的目标。
- **`Info.default(id)`**（schema/agent.ts:38-53）给出兜底形状：`mode:"primary"`、`hidden:false`、权限为"全 allow + .env 读 ask"四条——任何 `editor.update` 未曾注册过的 ID 都从这个默认值起步（core/agent.ts:63-64），这就是"注册表空但 update 即创建"的机制。

**配置侧形状**（`schema/src/config/agent.ts:10-21`）是 Info 的可选化投影（多一个 `disabled` 少一个 id/name——id 来自配置键名）。另有 V1 兼容层 `core/src/v1/config/agent.ts`（69 行）：接受旧字段 `prompt`/`tools`/`maxSteps`/`temperature`，`normalize`（L44-61）把 `tools: {bash: false}` 布尔开关翻译成 permission 规则、`maxSteps` 归并为 `steps`。

---

## 3. 注册管线与内置清单

### 3.1 插件 API

插件拿到的是 `AgentDomain`（plugin/src/effect/agent.ts:14-17）：

```ts
export interface AgentDomain extends AgentApi<unknown> {
  readonly transform: Transform<AgentEditor>   // 挂 editor 变更函数
  readonly reload: () => Effect.Effect<void>   // 触发注册表重建（配置文件改动后）
}
```

`AgentEditor`（同文件 L6-12）就是 core/agent.ts `Editor`（L32-38）的 DeepMutable 版：`list / get / default / update / remove`。**没有 add**——`update(id, fn)` 即 upsert（core/agent.ts:62-67：不存在则以 `Info.default(id)` 起步再 mutate）。每次 transform 完成后 `notify` 发 `Agent.Event.Updated`（ephemeral，core/agent.ts:72）。

### 3.2 内置 7 agent

注册处一：`plugin/agent.ts`（id `opencode.agent`，经 internal.ts:223 挂载）。6 个：

| id | mode | hidden | 特点（行号实测） |
|---|---|---|---|
| **build** | primary | 否 | 默认 agent（`defaultID`，core/agent.ts:13）。"执行工具按配置权限"（L80-85），只追加 `question * allow` 一条；其余继承 Info.default 的全 allow + env ask |
| **general** | subagent | 否 | 通用并行工兵（L87-96）：显式 deny `question` 与 `subagent`（不让子代理再问人/再嵌套），其余默认 allow |
| **explore** | subagent | 否 | 只读搜索专家（L98-118）：`system = PROMPT_EXPLORE`（L7-17，明令"EXCLUSIVELY search/analyze、shell 只读"）；权限先 `* * deny` 压底，再白名单 `shell/grep/glob/webfetch/websearch/read allow`，env 读 ask，`subagent deny`、`external_directory allow` |
| **compaction** | primary | 是 | 压缩摘要专用（L120-124），无自定义 prompt/权限——由 compaction.ts:214 按名取用 |
| **title** | primary | 是 | 标题生成器（L126-132）：`PROMPT_TITLE`（L19-62）+ `* * deny`（禁一切工具）；context.ts:96-107 还会替它挑小模型/低推理 variant |
| **summary** | primary | 是 | PR 风格总结（L134-140）：`PROMPT_SUMMARY` + `* * deny` |

注册处二：`plugin/plan.ts`（id `opencode.plan`，internal.ts:224）。第 7 个：

| **plan** | primary | 否 | 只读规划模式（L50-58）：`question allow` + `edit * deny` + `edit ~/.opencode/plan/* allow`——比 explore 宽（能问用户）但同样不能改代码；另带 enter/leave `<system-reminder>` 注入与越权报错美化（§4） |

注意三个 hidden primary（compaction/title/summary）是**内部机器代理**：它们不出现在可选列表（`selectable` 过滤 `mode!=="subagent" && !hidden`，core/agent.ts:74-75），但可被 `agents.get(ID)` 按名直接取用——内部子系统（title 生成、压缩、总结）绕过 UI 选择直接点名的暗道。

### 3.3 配置注册（第三条来源）

`config/plugin/agent.ts` 扫描配置目录 `{agent,agents,mode,modes}/**/*.md`（L21-24 legacySources）+ 配置文件 `agents:` 键（L104-131），V1 形状经 `ConfigAgentV1` 解码迁移。写入同样走 `ctx.agent.transform`：全局 `permissions` 逐条 push 到**所有现存 agent**（L102-103），再逐个 upsert 配置定义的 agent。文件 watcher（config.changes + `isAgentSource` L143-148）→ 100ms debounce（L83-85）→ `reload()` → `ctx.agent.reload()` 重建注册表。配置还可设 `default_agent`（L101）改默认选择。

### 3.4 默认选择链

`selectedDefault()`（core/agent.ts:76-86）三级回退：配置指定的 default（须 selectable）→ `build` → 注册表第一个 selectable。`list()`（L106-111）把默认 agent 排到首位——UI 列表顺序即默认优先。注意 `resolve(undefined)` 与 `select(undefined)` 都走这条链（core/agent.ts:94-105）：会话无 agent 字段时（老数据/未指定），"默认 agent"是解析期概念而非存储期概念。

### 3.3a 自定义 agent 的两条用户路径

```jsonc
// opencode.json：全局 agent 定义
{ "agents": {
    "reviewer": {
      "description": "Code review specialist",
      "mode": "subagent",
      "steps": 12,
      "permissions": [
        { "action": "edit", "resource": "*", "effect": "deny" }
      ]
    }
} }
```

```markdown
<!-- .opencode/agent/reviewer.md：目录式定义（frontmatter 即 ConfigAgent.Info） -->
---
description: Code review specialist
mode: subagent
steps: 12
---
You are a code reviewer…（正文进 system 字段）
```

两条路都汇入 config/plugin/agent.ts:104-131 的同一个 upsert 循环：`disabled:true` 则 remove（L108-110）；已存在的 agent（含内置）按字段覆盖——**配置可覆写内置 agent 的 model/system/steps，但 permissions 是 push 追加而非替换**（L113 `if (!exists)` 决定全局权限只加一次；L126-128 配置级权限同样 push）。

### 3.5 transform 的执行时机与宿主

`ctx.agent.transform(fn)` 挂载的函数在插件宿主 `PluginHost` 初始化时执行（plugin/host.ts:99 `listAgents`、L145-149 将 editor 暴露为 RPC 响应面）。注册表的完整生命周期：

```
Location 启动
  └─ internal.ts 内置插件逐个 init（顺序即数组顺序）
       ├─ AgentPlugin      transform → 注册 build/general/explore/compaction/title/summary
       ├─ PlanPlugin       transform → 注册 plan
       ├─ SubagentTool     注册 subagent 工具 + context hook（拼 agent 菜单）
       └─ ConfigAgentPlugin 初次 load() 扫描配置 → transform 应用（全局权限 push + 自定义 agent upsert）
  └─ Agent.Event.Updated（ephemeral，schema/agent.ts:11）→ TUI 刷新 agent 列表

配置文件变动（watcher + 100ms debounce）
  └─ ConfigAgentPlugin.reload → load() → ctx.agent.reload() → 全部 transform 重放
```

要点：`reload` 是**全量重建**而非增量——`ctx.agent.reload()`（core/agent.ts:90 暴露为 `state.reload`）会让宿主把所有已注册的 transform 函数按注册顺序重放一遍，配置插件的新数据在重放中生效。这保证了 transform 幂等性要求：所有内置注册都用 `update(id, fn)` 对固定字段赋值/push，天然可重放；但**权限 push 是追加语义**，重放前注册表已清空所以不会累积（`State.reload` 语义）。

### 3.6 server 面

HTTP 只读两个端点（server/handlers/agent.ts:10-26）：`agent.list`（直接 `Agent.Service.use(list)`，含 hidden）与 `agent.get`（404 = `AgentNotFoundError`）。**没有 agent.create/update RPC**——写路径只有插件 transform 一条。客户端模型层（packages/app/src/providers/models/agent.ts）消费 list 做 UI 状态。

---

## 4. 会话内切换

服务端实现极薄（session/session.ts:89-95）：

```ts
const switchAgent = Effect.fn("Session.switchAgent")(function* (sessionID, input) {
  const session = yield* get(sessionID)          // 只验证会话存在
  yield* bus.publish(SessionEvent.AgentSelected, // durable 事件，schema L69-77
    { sessionID, agent: input.agent, previous: session.agent })
})
```

**切换 = 一条 durable 事件**，不直接改任何内存态。三路投影消费：

1. **session 表**：projector.ts:555-563 把 `agent` 列更新为 `event.data.agent`（附 `time_updated`）——持久化的就这一个字段。
2. **消息流**：message-updater.ts:91-100 追加一条 `agent-switched` 消息（`SessionMessage.AgentSelected`，含 previous），用户在 TUI 里看到的"已切换到 Plan"就是它。
3. **Plan 模式提醒**（插件级）：plan.ts:92-113 订阅 `session.created` / `session.agent.selected`，进入 plan 注入 enter `<system-reminder>`、离开注入 leave——以 synthetic 消息持久化（失败仅 logWarning 不阻断，L106-109）。且有**对账逻辑**（L74-90 context hook）：压缩/revert 可能剥掉提醒，逐请求检查最后一条提醒是否与当前 agent 匹配，不匹配就在尾部（用户 prompt 前）补插，兼顾 provider 缓存前缀保温。

**不校验 agent 存在**：切到未注册 ID 事件照发；下一次 `SessionContext.select` 在 context.ts:129 才炸 `AgentNotFoundError`。切换本身永远成功——错误延迟到执行边界（见 §9）。special 场景：subagent 工具续用旧子会话但换了 agent 名时也会调 `switchAgent`（subagent.ts:175）。

### 4.1 切换的完整事件流（实测行号串接）

```
客户端 switchAgent(sessionID, "plan")
  │ session/session.ts:89-94  get() 校验会话存在 → bus.publish(AgentSelected)
  ▼
Bus（durable，session-event.ts:69-77：{agent, previous?}）
  ├─ projector.ts:555-563   UPDATE session SET agent='plan', time_updated=… （事务内先 run 事件再更新）
  ├─ message-updater.ts:91-100  appendMessage(agent-switched, {agent, previous})
  ├─ plan.ts:92-113（插件订阅） enter/leave reminder → session.synthetic() 持久化为 user 消息
  └─ TUI 实时流刷新 agent 显示
  ▼
下一次 prompt → runner.drain → context.select → agents.select("plan") → 新行为包生效
```

`switchModel`（session.ts:96-108）是同构兄弟：同样先查重（同模型直接 return）再发 `ModelSelected`。subagent 场景两者并发调用（subagent.ts:170-178 `Effect.all`）。

### 4.2 会话创建时的 agent 锚定

`Session.create`（session.ts:262-300）把 `input.agent` 直接写进 `session.created` 事件（L284），projector 落入 session 表 agent 列（projector.ts:460）。fork 子会话继承 `parent.agent`（store.ts:169）。即：**agent 归属从创建那一刻就是 durable 事实**，switch 只是后续修订。

---

## 5. Step Allowance

每个 agent 可配 `steps`（正整数）。语义链（三处实测）：

1. **判定**：runner/llm.ts:226 `const stepLimitReached = loaded.agent.info.steps !== undefined && step >= loaded.agent.info.steps`——**未配置即无上限**。
2. **软着陆而非硬中断**：到限后不是报错结束，而是把 `MAX_STEPS_PROMPT`（max-steps.ts:1-16，"工具已禁用、只准文字总结"）作为 assistant 消息追加进 transcript，且 `toolChoice: "none"` 但**保留工具定义**以保温 provider 缓存前缀（llm.ts:227-238 注释明言）。
3. **双保险**：即便模型仍试图调工具，step.ts:92-94 在 `toolChoice.type === "none"` 时直接返回 `Tool.Error("Tools are disabled after the maximum agent steps")`。

**计数与重置**：step 计数属于 runner drain 循环（llm.ts:61 起步 `input.continuation?.step ?? 1`）；新用户输入提升后 `step = 1`（llm.ts:111），steer 提升 `promoted > 0` 时同样重置（llm.ts:181）。AGENTS.md L185 的措辞即此："Promoting new user input resets the selected agent's step allowance; a batch of steers resets it once"——**换 agent 不重置 step 计数**（计数在会话维度），但上限随新 agent 的 `steps` 字段变化（每步实时读 `loaded.agent.info.steps`）。

`runner/model.ts` 与 agent 的关系：模型解析**不直接消费 agent**——`resolve(session, available)`（L75-91）只看 `session.model`，无则全局 `ModelResolver.resolve()`。agent 的模型偏好在**会话创建/切换时**就已写进 `session.model`（subagent.ts:189 `override ?? agent.model ?? parent.model`）。即 agent→模型的解析发生在入口（create/subagent），不在 runner。

### 5.1 agent.system 进入请求的位置

`baseTranscript`（model-request.ts:100-118）是 agent 定义进入 LLM 请求的唯一入口：

```ts
system: [
  input.agent.system                       // agent 自带 prompt 优先
    ? input.agent.system
    : SessionSystemPrompt.make(            // 否则 harness prompt（runner/prompt/system.txt）
        input.tools.definitions.map((t) => t.name)),  // 按可用工具注入工具指南
  input.initial,                           // 指令基线（instructions 子系统）
].filter(nonEmpty).map(SystemPart.make)
```

注意 system prompt 组装细节归 [6_context.md](6_context.md)；此处只标注接缝——**agent.system 是全有或全无的替换**，不与 harness prompt 拼接（explore/title/summary 完全替换，build/plan 用 harness + 权限差异）。generate 路径（标题/总结类一次性生成，session/generate.ts:44-46）复用同一个 baseTranscript。

### 5.2 每个内置 agent 的 steps 现状

内置 7 个均未设 `steps`（plugin/agent.ts 与 plan.ts 的 update 块里都没有 `item.steps` 赋值）——上限是**用户配置侧维度**（ConfigAgent.steps，config/plugin/agent.ts:120 透传）。典型用途：给自定义 subagent 设小步数防止失控。

---

## 6. Subagent 接缝（细节留 12_subagent 篇）

subagent 工具（tool/plugin/subagent.ts）是 agent 定义的**唯一运行时消费者之外的最大出口**：

- **解析与校验**：`agents.resolve(input.agent)` 不存在→`Unknown agent`（L139-140）；`mode === "primary"`→拒绝（L148-149）；权限评估 `action:"subagent", resource:agent.id`（L150-165）——所以 general/explore 的 `subagent * deny` 使它们不能再嵌套。
- **派生**：`sessions.create({ parentID, agent: agent.id, model: override ?? agent.model ?? parent.model })`（L187-196）——**子会话生来带 agent ID**，后续执行路径与主会话完全同构（SessionContext.select 同一套）。
- **工具描述动态注入**（L284-303，context/compaction/generate 三个 hook）：每次组装上下文时，把所有 `mode !== "primary" && !hidden` 且当前 agent 权限不 deny 的 agent，以 `- {id}: {description}` 清单拼进 subagent 工具的 description——**description 字段的真实用途在这**：它是父 agent 的路由菜单。

深度限制：沿 parentID 链数深度，超过 `experimental.subagent_depth`（默认 1，L129-133）报错。

### 6.1 子会话与主会话的 agent 语义差异

派生出的子会话在数据层与主会话**完全同构**（同一张 session 表、同一个 runner），仅有的差异注入点：

- 首条 prompt 前缀 `"You are a subagent spawned by another session."`（subagent.ts:216-218，仅新建时）；
- `parentID` 非空 → 无标题生成（llm.ts:178 `!selected.session.parentID` 才起 title 任务）；
- 完成结算走 SubagentJob/SubagentCompletion（recovery 记录 L221-227，含 agent 名与描述）——细节全部留 12_subagent 篇。

续聊校验：`existing.parentID !== context.sessionID` 拒绝（L159-161）——子会话只能被生父续用；换 agent 续聊 = switchAgent + （switched 时）重设 agent.model（L170-176 注释：显式 model > agent 配置 model > 继承）。

---

## 7. 权限内嵌（细节留 11_permission 篇）

agent.permissions 进入权限评估的两个汇合点：

1. **工具快照收窄**：context.ts:131 `Permission.merge(agent.info.permissions, session.permissions ?? [])` → `registry.snapshot(permissions)`——agent 权限直接决定**模型看得见哪些工具**（skills/MCP 工具同此收窄，L134-136）。这是"行为包"中工具集维度的实现：**不是列白名单，而是权限过滤**。
2. **运行时断言**：permission.ts:158-163 `configured()` 每次评估 `merge(agent?.permissions ?? missingAgentPermissions, session.permissions)`——注意 `missingAgentPermissions = [{action:"*",resource:"*",effect:"deny"}]`（L19）：**agent 解析失败的会话默认全禁**，fail-closed。

合并语义：session 权限**追加在 agent 权限之后**（数组序 = findLast 优先序），即会话级规则可覆盖 agent 默认。managed-policy（组织策略）走的是另一条路（config policy，作用于配置层），与 agent.permissions 正交。

---

## 8. 不变量与设计取舍

从代码与 AGENTS.md 提取：

1. **注册表永不为空是插件的职责，不是核心的**——core/agent.ts 116 行里没有一行硬编码 agent；`build` 也只是 `defaultID` 常量 + selectedDefault 的回退名。核心只提供"空 Map + upsert editor + 选择回退链"。取舍：内置行为与核心彻底解耦，internal.ts 的挂载顺序（ConfigAgentPlugin L255 在 AgentPlugin L223 之后）决定配置能否覆盖内置（后写胜）。
2. **会话持久化只存 ID，不快照定义**（projector 只写 agent 列一个字符串）。取舍：agent 定义热重载后旧会话立即生效新行为；代价是定义消失即 AgentNotFoundError（§9）。
3. **update 即 upsert、default 即默认**——没有 add/remove 对称 API；`Info.default(id)` 的宽容默认（全 allow）使任何插件一行 `editor.update` 就能造出可用 agent。
4. **hidden primary = 内部机器代理暗道**：title/compaction/summary 不经 UI 选择、被子系统按名直取；它们 `* * deny` 的权限保证机器代理绝不产生副作用。
5. **权限数组语义靠序**：deny 压底 + 白名单 allow 是惯用法（explore/Info.default）；findLast 评估让"后写胜"贯穿 agent 内规则、agent↔session 合并、managed-policy 三层。
6. **steps 用软着陆**（prompt 注入 + toolChoice none）而非硬错误，且保留工具定义保温缓存（llm.ts:236-237 注释）——宁可让模型自己总结也不浪费已建立的 provider 缓存前缀。
7. **切换不前置校验**：AgentSelected 事件先行，存在性校验延迟到执行边界（select）。事件溯源的典型取舍——命令路径尽量薄，失败留给读侧。

---

## 9. 失败与恢复

1. **Agent 不存在（执行时）**：`SessionContext.select` → `AgentNotFoundError`（context.ts:129，error.ts:36 定义）；to-session-error.ts:73 映射为 `{type:"unknown", message}`——会话进入 error 态，用户可切回存在的 agent 恢复（切换本身不校验，见 §4）。runner 的错误联合类型显式收录它（runner/index.ts:8、16；generate.ts:19），即一次坏切换最多损失一次 run，不损坏会话数据。
2. **Agent 不存在（subagent 调用）**：`Unknown agent: {id}` ToolFailure（subagent.ts:140），父会话不受影响，模型可换 agent 重试。
3. **Agent 不存在（HTTP）**：server/handlers/agent.ts:19-23 `AgentNotFoundError` 404。
4. **权限缺失的会话**：fail-closed 全禁（permission.ts:19、162）——比放行安全。
5. **切换中断**：AgentSelected 是 durable 事件，进程崩溃后投影已落库（或随事件重放落库）；不存在"切了一半"。Plan 提醒丢失有对账路径（plan.ts:74-90 每 request 检查补插）；提醒写入失败只 logWarning（L86-89）降级为"上下文里有但历史里没有"，不阻断主流程。
6. **注册表热重载竞态**：ConfigAgentPlugin 的 watcher 订阅先于初次扫描（agent config 插件 L68-73 注释），保证 scan 与 update 竞态也能触发重建——重载窗口内正在执行的 Step 用的是已加载的 `loaded.agent` 快照，不受影响。

---

## 演进注记（对照旧笔记 4.4，2026-08-03 基线）

### 漂移总览

1. **"空注册表+插件注入"判断成立且更彻底**（★★★）：旧记 4.4 记录 core/src/agent.ts 111 行空注册表；现 116 行，结构未变（State.Transformable + Editor），`selectable`/`selectedDefault`/list 排序为成熟后的细节。V1 的 `opencode/src/agent/agent.ts`（453 行硬编码 7 agent）随 `packages/opencode/src` 整体退役，`packages/core/src/v1/` 只剩**配置 schema 兼容层**（v1/config/agent.ts 69 行，负责翻译旧 `tools`/`maxSteps` 字段）。
2. **内置清单从"核心硬编码"变为"两个内置插件"**（★★★）：7 个内置 agent 分家——6 个在 `plugin/agent.ts`（id `opencode.agent`），plan 独立成 `plugin/plan.ts`，因为 plan 需要事件订阅 + context hook 的完整插件能力（切换提醒/对账），不只是注册。V1 时代 plan 只是 7 个硬编码之一。
3. **字段维度**（★★）：V1 的 `prompt` → `system`；`tools:{name:boolean}` → `permissions` 规则集（V1 兼容层仍在翻译）；新增 `steps`（吸收旧 `maxSteps`）、`request`（provider 参数下沉到 agent）、`mode:"all"`。V1 的 Color 枚举（primary/success/…）收敛为自由字符串。
4. **会话侧锚点变化**（★★）：切换从（旧记）直接改会话状态，演进为 `AgentSelected` durable 事件 + projector 写列 + message-updater 追加 agent-switched 消息——与 v2 事件溯源改造同构（详见 4_session.md 演进注记）。
5. **step 上限从硬停到软着陆**（★）：MAX_STEPS_PROMPT + toolChoice:"none" + 缓存保温组合为现行实现；step 重置语义（新输入/steer 批重置一次）写入 AGENTS.md L185。

## 相关篇

- [4_session.md](4_session.md)：Session 聚合与 runner——agent 在其中的位置（select/load/drain）
- [9_tool.md](9_tool.md)：Tool 注册表与 `registry.snapshot(permissions)`——agent 权限如何收窄工具
- [6_context.md](6_context.md)：system prompt 组装（baseTranscript / SystemPrompt.make）与 instructions 子系统
- 未来 11_permission 篇：Permission.Service 评估链、merge 语义、ask/deferred 流程
- 未来 12_subagent 篇：subagent 工具全流程（Job/前台后台/SubagentJob 恢复）
