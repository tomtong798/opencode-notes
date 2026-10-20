# 上下文代数详解（Instructions Source 代数：一次 LLM 调用的上下文 = system prompt + 指令基线 + chronological 指令增量 + durable transcript）

> 一句话：**上下文子系统把"模型该知道什么"建模为一组可组合的 Instructions Source（每个 source 有 key/read/render，值经 canonical JSON 哈希去重后落三张 SQLite 表），`SessionContext.select/load` 两阶段解析出 system prompt + 指令基线 + 消息重放，指令变更以 delta 事件按时间序注入，epoch 由 Compaction.Ended 投影推进。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19，`wc -l` + `cat -n`/`sed -n` 抽查）。
> 权威出处：无独立 spec 篇；对应 `AGENTS.md` 相关条目与 `specs/v2/session.md`（转引自 [4_session.md](4_session.md)）。
> [返回 README](README.md) | 前置 [4_session.md](4_session.md)（§6 读取路径、inbox/promote 时序） | 关联 [9_tool.md](9_tool.md)（tool snapshot）；agent 定义细节归未来 8_agent 篇（互链占位）；Step 组装时序归 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)。

---

## 0. 模块地图

两个 instructions 位置并存，职责不同（均已实测确认，`ls` 实查）：

- `packages/core/src/instructions/`——**纯代数库**：Source 类型、make/combine/read/diff/render、哈希；不 import 数据库。
- `packages/core/src/session/instruction-*.ts`——**会话侧持久化与应用**：三表读写、observe/commit/apply/epoch。
- `packages/core/src/instruction-discovery.ts`（core 根）——**发现服务**：AGENTS.md 等文件的可变状态容器（真正的扫描在 `config/plugin/instruction.ts` 插件里）。

旧笔记的 `system-context/` 目录**已不存在**（`ls packages/core/src | grep system-context` 为空）；`context-epoch.ts` 也已删除（见演进注记）。

```
packages/core/src/
├─ instruction-discovery.ts ★ 发现服务：State<Editor> 容器 + list/load（149 行）
│    （实际文件扫描/监听在 config/plugin/instruction.ts 插件，约 220 行，见 §3）
├─ instructions/                     ★ 指令代数（纯函数库）
│   ├─ index.ts        Source/make/combine/read/diff/render/hash（256 行）
│   └─ builtins.ts     core/date + core/environment 两个内建源（60 行）
└─ session/
    ├─ context.ts        ★ SessionContext：select/load 两阶段（208 行）
    ├─ system-prompt.ts  默认 system 模板 + 工具指导渲染（28 行）
    │                      + runner/prompt/system.txt 模板本体（15 行）
    ├─ instructions.ts   SessionInstructions：@-提及指令文件的合成消息注入（125 行）
    ├─ instruction-state.ts ★ state/blob 两表读写 + observe/commit/epoch（274 行）
    ├─ instruction-entry.ts  instruction_entry 表的 KV 上下文 API（181 行）
    ├─ provider-context.ts   compaction 原生检查点的 provenance/编解码（84 行）
    ├─ environment.ts     SessionEnvironment：进程内每会话 shell 变量（33 行）
    ├─ sql.ts             三表 DDL：instruction_entry(147)/instruction_blob(162)/instruction_state(167)
    ├─ model-request.ts   baseTranscript：system 数组 + messages 组装（105-119）
    ├─ history.ts         entriesForRunner：initial + 消息重放（115-133）
    └─ runner/llm.ts      prepareContext：Step 前 InstructionState.prepare（202-207）
```

`wc -l` 实测：

```
$ wc -l packages/core/src/session/context.ts system-prompt.ts instructions.ts \
       instruction-state.ts instruction-entry.ts ../instruction-discovery.ts \
       ../instructions/{index,builtins}.ts provider-context.ts environment.ts
   208  session/context.ts
    28  session/system-prompt.ts
   125  session/instructions.ts
   274  session/instruction-state.ts
   181  session/instruction-entry.ts
   149  instruction-discovery.ts
    60  instructions/builtins.ts
   256  instructions/index.ts
    84  session/provider-context.ts
    33  session/environment.ts
 1398  total
```

## 1. 心智模型

一次 LLM 调用的完整上下文（`SessionModelRequest.baseTranscript`，model-request.ts:105-119）：

```
LLM 请求
├─ system: [SystemPart, SystemPart]           ← model-request.ts:110-117
│   ├─ [0] agent.system（自定义）              ← agent 定义（8_agent 篇）
│   │        或 SessionSystemPrompt.make()     ← 默认 harness 模板 + 工具指导（§2）
│   └─ [1] initial                             ← 指令 epoch 基线渲染文本（§4/§5）
│              = renderInitial(all sources, initial_values 解引用后)
└─ messages: durable 消息重放                  ← 4_session.md §6 读取路径
    └─ 含 chronological 的 InstructionsUpdated 渲染文本
       （每次指令变更以事件文本进入对话流，模型"看到"日期变了、AGENTS.md 改了）
```

指令不是每次全量重发：**epoch 基线**（system[1]）+ **增量文本**（消息流里）的分片方式，使 prompt 前缀保持可缓存（context.ts:147-149 注释明确此目标）。

```
Instructions Source 代数（instructions/index.ts）
┌────────────────────────────────────────────────────────────┐
│ Source = { key, read: Effect<Json | Unavailable | Removed>, │
│            initial(value), changed(prev, cur), removed(p) } │
│ make(def)      类型化定义 → 闭包 Source（index.ts:237-271）  │
│ combine(lists) 拼接 + DuplicateKeyError（:273-281）          │
│ read(list)     并发读所有源 → ReadResult（:283-289）         │
│ diff(observed, prev) → Admission{delta, blobs}（:291-308）   │
│ hash(value)    canonical JSON → sha256（:343-345, :392-400） │
│ renderInitial / renderUpdate  按 key 序渲染（:310-341）       │
└────────────────────────────────────────────────────────────┘
```

Source 接口原文（index.ts:182-188）：

```ts
export interface Source {
  readonly key: Key
  readonly read: Effect.Effect<Schema.Json | Unavailable | Removed>
  readonly initial: (value: Schema.Json) => string | undefined
  readonly changed: (previous: Schema.Json, current: Schema.Json) => string | undefined
  readonly removed: (previous: Schema.Json) => string | undefined
}
```

三种"非值"（index.ts:168-174，引用相等判别 :384-390——"identity, never structure, discriminates"）：

- `Unavailable`——读暂时失败，存储值继续生效（不产生 delta，index.ts:297 跳过）；
- `Removed`——源存在但值没了（AGENTS.md 被删），产生 `removed` delta（:299）；
- 无 previous 时的 Unavailable → `InitializationBlocked`（:292-293）——首次基线不允许建立在失败读取上。

## 2. System Prompt 组装

system-prompt.ts 全文仅 28 行。模板本体 `runner/prompt/system.txt`（15 行）开头实测：

```
$ sed -n '1,8p' packages/core/src/session/runner/prompt/system.txt
You are an AI agent running in OpenCode, a coding agent harness. ...
# Harness
- Responses are rendered as GitHub-flavored Markdown.
- `<system-reminder>` blocks are harness instructions, not user-authored content. ...
- Prefer parallelizing independent tool calls.
${OPENCODE_TOOL_GUIDANCE}
```

`make(tools)`（system-prompt.ts:5-7）委托 `render`（:9-27）：按工具名条件注入指导——`shell` → "Prefer dedicated tools over shell commands..." 等 2 条（:11-16）；`write` 1 条（:18-21）；`edit` 1 条（oldString/newString 语义，:23-26）——再做 `${OPENCODE_TOOL_GUIDANCE}` 占位替换（:27）。

一条增量在消息流里的样子（示意，渲染文案取自实测源码）：用户在会话中途编辑了 `AGENTS.md` → 下一次 `prepareContext` 的 `observe` 发现 `core/instructions` 的 hash 变了 → `commit` 发布 `InstructionsUpdated`，事件文本为 "The instructions from /path/AGENTS.md changed. Here's the diff: ```diff ...```" → 投影落消息行，模型在后续请求的 messages 里看到这段文本。日期跨天、`<env>` 变化、api context 更新（"The context under \"key\" changed and supersedes the previous value:"，entry.ts:92-94）同理。

**关键纠偏（对照直觉）**：时间/环境注入**不在** system prompt 里，而是两个内建指令源（builtins.ts）：

- `core/date`（builtins.ts:23-30）：`read: DateTime.nowAsDate → toDateString`，initial 渲染 `Today's date: <date>`，changed 渲染 `Today's date is now: <date>`（跨天会话在消息流里产生日期增量，而非重发基线）；
- `core/environment`（:32-47）：`<env>` 块——working directory / workspace root / git repo / platform / 专用 tmp 目录建议，initial 前缀 "Here is some useful information about the environment you are running in:"，changed 前缀 "The environment you are running in is now:"。

```ts
// builtins.ts:43-50（sed -n 实测）
"<env>",
`  Working directory: ${location.directory}`,
`  Workspace root folder: ${location.project.directory}`,
`  Is directory a git repo: ${location.vcs?.type === "git" ? "yes" : "no"}`,
`  Platform: ${process.platform}`,
`  Prefer ${global.tmp} over generic system temporary directories such as /tmp; ...`,
"</env>",
```

agent 优先级（model-request.ts:111-113）：`agent.system` 非空则**完全替换**默认模板（不是拼接）；插件侧 optimize.ts:84 复用 `render` 做工具变更时的增量指导。最终 system 数组 = `[agent 定义或默认模板, initial 指令基线]`，空段过滤后各包成 `SystemPart.make`（:116-117）。

## 3. 指令发现链

两跳结构：**扫描插件写状态容器，状态容器闭包成单一 Source。**

### 3.1 第一跳——扫描插件（config/plugin/instruction.ts）

Plugin id `opencode.config.instruction`。候选集合（:51-58 实测）：

```ts
// config/plugin/instruction.ts:51-58（实测）
const candidates = [
  ...(discovery.global ? [globalFile] : []),          // ~/.config/opencode/AGENTS.md
  ...(project
    ? ancestorDirectories(start, stop)
        .map((directory) => join(directory, "AGENTS.md"))
        .filter((file) => discovery.global || file !== globalFile)
    : []),
]
```

即路径层级为：**全局 AGENTS.md + 从工作目录（start）向上到项目根（stop）的每级祖先目录里的 AGENTS.md**（子目录 → 根，近者优先入列）。`stop` 规则（:40）：start 在 home 下则止于 home，否则止于 project root；`project` 仅当 start 在 root 内（:39）。

三路来源隔离读取（refresh，:169-191 实测）：`global` / `project` / `configured`：

- `configured`（:104-130）= config `instructions` 数组：相对模式 `globUp(start→root)`（:97）、绝对路径扫描、`~/` 展开（:91）、URL 远程指令（:108-110）；
- 任一路失败仅该路 `unavailable`（`isolate`，:161-168）；**三者全可用才整体 available**，否则整体 `Unavailable`（refresh 末段三元判断）。

文件 watcher 订阅全部候选（:59-62）+ 100ms 防抖重扫（:185-190 注释："Editor saves arrive as bursts of watcher events; settle before rescanning once"）；`config.updated` 事件触发 reconfigure+refresh+reload。远程 URL 仅 config 加载时抓取（5s 超时，失败保留上次副本，:132-134 注释："an unreachable URL never makes the whole instruction set unavailable"）。

### 3.2 第二跳——状态容器（instruction-discovery.ts）

插件通过 `discovery.transform(editor => ...)` 写入 `State<Data, Editor>`（:60-78）；`list()`（:92-96）返回文件数组或 `unavailable`；`load()`（:104-108）闭包成单一 Source（key `core/instructions`，:17）：空文件集 → `removed`（:107）——"没有 AGENTS.md"本身就是一种要告知模型的缺失。

渲染器实测（:123-149）：initial 拼接所有 `Instructions from: <path>\n<content>`；changed 用 `diffByKey` 按 path 对比，改动文件渲染 **unified diff**（context 3）：

```ts
// instruction-discovery.ts:138-146（实测）
const patch = createPatch(after.path, before.content, after.content, "", "", { context: 3 })
...
const replacement = `The instructions changed:\n${render([after])}`
return diff.length < replacement.length ? diff : replacement
```

`Options {project, global}`（:47-51）默认都开；注释（:39-40）解释策略放层选项的原因："internal plugins have no per-composition options channel"。

### 3.3 兄弟源汇聚

`SessionContext.select`（context.ts:150-158）按"最共享在前"合并七类源（注释 :147-149）：`CodeModeInstructions`（工具目录）→ MCP 指令 → references → skills → **discovery（AGENTS.md）** → **builtins（date/env）** → **entries（api 上下文 KV）**。

另注意与 `session/instructions.ts` 的分工：那是**用户 @-提及指令文件**的注入通道（见 §10），不参与发现链。

## 4. 指令状态三表

sql.ts DDL 实测：

```
instruction_entry (sql.ts:147-160)      instruction_blob (:162-165)   instruction_state (:167-175)
├─ session_id → session(id) 级联删除    ├─ hash PK（内容寻址）          ├─ session_id PK → session(id)
├─ key  ┐ 复合主键 (session_id,key)    └─ value (JSON)                ├─ epoch_start  ┐ epoch 语义（§5）
├─ value (JSON, nullable)                                             ├─ through_seq  ┘
├─ removed bool, default false                                        ├─ initial_values  (key→hash)
└─ time_created / time_updated                                        └─ current_values  (key→hash)
```

### 4.1 instruction_entry——API 侧会话 KV 上下文

instruction-entry.ts `Interface`（:69-79）：`put/remove/list/load`。put 有 **256KiB 硬限**：

```ts
// instruction-entry.ts:135-141（实测）；MaxValueBytes = 256*1024（schema/instruction-entry.ts:30）
const actualBytes = Buffer.byteLength(JSON.stringify(input.value), "utf8")
if (actualBytes > MaxValueBytes)
  yield* new ValueTooLargeError({ actualBytes, maxBytes: MaxValueBytes, ... })
```

`load`（:173-175）每行（含 removed 行）产出一个 Source，key 加前缀 `api/<key>`（:91），渲染：

```ts
// instruction-entry.ts:85-88（实测）
const renderBlock = (key, value) => [`<context key="${key}">`, renderValue(value), "</context>"].join("\n")
```

注释（:82-84）明确设计意图："Rendering stays mechanism-neutral: the model sees session context, not how it was attached."

### 4.2 instruction_blob——内容寻址存储

hash → JSON，同一份 AGENTS.md 内容跨会话/跨 epoch 只存一份。`insertBlobs` 用 `onConflictDoNothing`（state.ts:227-231）；读取按 500 hash 一批、并发 4（:233-243）；缺 blob 直接 `Effect.die`（:245-247）——内容寻址下 blob 缺失是不可恢复的存储损坏。

### 4.3 instruction_state——每会话一行

`initial_values`（epoch 基线引用）与 `current_values`（最新引用），读写见 §5。

### 4.4 复用与失效总结

- **复用**：blob append-only；state 在指令内容 hash 不变时跳过（index.ts:303 `previous?.[entry.key] === next` → continue）；
- **失效**：即 delta——新 hash 覆盖或 `removed`（index.ts:294-306）；`applyHashDelta`（:347-354）把 `removed` 从 Values 中删除键。

## 5. 会话上下文与重建语义

`SessionContext`（context.ts）两阶段（Interface 注释 :45-50 实测）："`select` fixes the Session, agent, instruction sources, and tool snapshot; `load` adds the model and active history for that selection."

**"上下文变了需要重建"的现代表达**（旧 Context Epoch 文件已删，语义收进三表 + 事件流）：

```
每次 Step 前（llm.ts:202-207 prepareContext）
├─ context.select(sessionID)            ← 七类指令源 + 工具快照（context.ts:121-161）
│    └─ 权限收窄：Permission.merge(agent, session) 同收窄 discovery 与 tools（:130-131 注释）
├─ InstructionState.prepare             ← state.ts:80-87 = observe + commit
│    ├─ observe（:21-36）：并发 read 全部源 + 查 state 行 → diff → {delta, blobs, initial?}
│    ├─ commit（:38-61）：无变更且非初始 → 跳过（:44）
│    │    否则 publish SessionEvent.InstructionsUpdated（durable）
│    │    ├─ 渲染文本冻结进事件（:46-47 注释）
│    │    └─ initial 同步打 metadata.instructions.initial=true（:56-57）
│    └─ 事件 commit 回调 insertBlobs（:58）
├─ 投影（projector.ts:683-688）：写消息行 + InstructionState.apply
│    └─ apply（state.ts:89-117）：current_values ← applyHashDelta(delta)，through_seq=seq
└─ context.load（context.ts:163-179）
     └─ SessionHistory.entriesForRunner（history.ts:115-133，事务内）
          └─ InstructionState.initial（state.ts:165-174）：解引用 initial_values 全部 blob
             → renderInitial → 基线文本；state 行缺失 → Effect.die（:171）
```

**epoch 推进**：`Compaction.Ended` 投影调 `advanceEpoch`（projector.ts:733；state.ts:139-154 实测）：

```ts
// instruction-state.ts:145-150（实测）
.set({
  epoch_start: epochStart,        // = compaction 的 durable seq
  through_seq: epochStart,
  initial_values: sql`${InstructionStateTable.current_values}`,
})
```

即压缩点之后，旧的时间序指令增量文本被基线吸收（history.ts:100-105 注释："the baseline supersedes the chronological updates before it"）。

一个具体时间线（行号均实测）：

```
seq 1  InstructionState.initialize（projector.ts:234）
        initial_values = current_values = {core/date:h1, core/instructions:h2, ...}
seq 5  AGENTS.md 编辑 → prepare → observe 发现 hash h2→h3
        InstructionsUpdated{delta:{core/instructions:h3}, blobs:{h3:...}, text:"...diff..."}
        apply：current_values[core/instructions]=h3，through_seq=5
seq 9  Compaction.Ended → advanceEpoch(9)
        epoch_start=through_seq=9，initial_values ← current_values（h3 吸收进基线）
seq 12 revert commit → reset（projector.ts:793）删行 → 下次 prepare 重建 initial
```

**其他生命周期**：revert commit 投影 `InstructionState.reset` 删 state 行（projector.ts:793；state.ts:156-162），下次 prepare 重建初始基线；fork 用 `InstructionState.current`（:177-182，注释 "used to seed a fork's baseline"）播种父会话当前值（session.ts:344）；`preview`（:184-208）为手动 compaction 提供不入库的 initial+update 双文本预览。

## 6. Provider 上下文调整

**实测纠偏**：provider-context.ts **不是** Claude cache 断点，而是 **compaction 原生检查点的 provenance 与编解码**：

```ts
// provider-context.ts:19-30（实测）
export function provenance(resolved): Provenance | undefined {
  const endpoint = resolved.model.route.endpoint
  if (!endpoint.baseURL || typeof endpoint.path !== "string") return undefined
  return { providerID, provider, modelID, route, protocol,
           endpoint: Hash.sha256(JSON.stringify([baseURL, path, sortedQuery])) }
}
```

- 注释（:17）："No guessed endpoints. Dynamic URL builders cannot establish a durable deployment identity here."——只认静态 baseURL/path；
- `compatible(source, target)`（:39-41）深比较：同一 provider 端点的会话才能重放对方的原生检查点（context.ts:169 把 provenance（或 `"local"`）作为 boundary 传给 `entriesForRunner`，即 4_session §6 的 compaction 边界参数）；
- `isCheckpoint`（:43-46）：`compaction && completed && providerContext !== undefined`——装了原生替换窗口（而非本地摘要）的完成压缩；
- `encode/decode/validate`（:52-56）：检查点存**规范替换消息**（注释："Stores the canonical replacement, not a local summary or transport continuation"；`Media.Asset.toJSON` 保 base64）；
- `upgradeLegacyMedia`（:64-82）：兼容 2.0.15 前旧 media 部件格式（mediaType/data 直挂 part → `media.source`）的读时迁移。

per-provider 的实际调整（缓存前缀等）在 transport/plugin 层，不在本文件。

## 7. Transcript 组装接缝

`baseTranscript`（model-request.ts:105-119）是上下文代数与执行链的唯一接缝：

```
baseTranscript({agent, model, tools, initial, messages})
├─ providerMetadataKey = route.providerMetadataKey ?? model.provider   ← :108
├─ system = [agent.system ?? SessionSystemPrompt.make(tool names),     ← :110-113
│            initial].filter(len>0).map(SystemPart.make)              ← :116-117
└─ messages = toLLMMessages(messages, model.ref, providerMetadataKey) ← :118（runner/to-llm-message.ts）
```

```ts
// model-request.ts:110-118（实测）
system: [
  input.agent.system
    ? input.agent.system
    : SessionSystemPrompt.make(input.tools.definitions.map((tool) => tool.name)),
  input.initial,
]
  .filter((part) => part.length > 0)
  .map(SystemPart.make),
messages: toLLMMessages(input.messages, input.model.ref, providerMetadataKey),
```

调用点：llm.ts `runStep` 组装 transcript 后交 `context.request.primary`；达到 step 上限时追加 MAX_STEPS_PROMPT assistant 消息且 `toolChoice:"none"`，注释："Keep tool definitions on the final Step to preserve the provider's cached prefix."。手动 compaction 走 `SessionHistory.preview` 的 initial+settled 消息。

durable 消息如何被 `messageEntries` 裁剪（native checkpoint 前的 system 消息过滤、replayable 判定）归 [4_session.md](4_session.md) §6，此处不重复；Step 内流式/工具时序归 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)。

## 8. 环境快照

session/environment.ts（33 行）极简，全文结构：

```ts
// environment.ts:17-28（实测）
const environments = new Map<SessionSchema.ID, Variables>()
return Service.of({
  get: (sessionID) => Effect.sync(() => environments.get(sessionID)),
  set: (sessionID, variables) => Effect.sync(() => { environments.set(sessionID, { ...variables }) }),
  clear: (sessionID) => Effect.sync(() => { environments.delete(sessionID) }),
})
```

`makeGlobalNode`（:30）——跨 Location 共享、**非持久**（进程内存）。用途：session.ts:129-130/256 暴露 API 读写每会话变量；shell.ts:262 执行命令时以 `environments.get(sessionID) ?? process.env` 作为子进程环境。即"会话环境快照"只影响 shell 工具进程环境，**不进入** LLM 上下文——进入 LLM 的是 builtins 的 `core/environment` 静态 `<env>` 块（§2）。

## 9. 不变量与设计取舍

从代码注释直接提取（均为实测注释，注明出处）：

1. **前缀可缓存**（context.ts:147-149）："Ordered from most to least shared across sessions so the baseline stays a reusable prompt-cache prefix"——catalog → 项目 → 日期/环境；llm.ts 同理把工具定义留在最后一个 Step。
2. **canonical JSON 即身份**（index.ts:176-181）："the same representation that is hashed, stored, and replayed"；`combine` 拒绝重复 key（:277 DuplicateKeyError）——key 是代数命名空间（`core/*`、`api/*`、插件自有）。
3. **渲染文本冻结**（state.ts:45-47）："re-rendering it later would require the original Location-scoped instruction sources"——增量文本写入 durable 事件。
4. **基线非历史**（state.ts:56-57）："Initial sync establishes the baseline; unlike later deltas it is not chronological history."
5. **Unavailable ≠ Removed**（index.ts:168-174）："The read failed temporarily; the stored value stands" vs "An observed absence"；首次基线遇 Unavailable 阻塞（InitializationBlocked，:217-224）而非带病建立。
6. **渲染机制中立**（entry.ts:82-84）：模型看到上下文内容，看不到附加机制。
7. **不猜端点**（provider-context.ts:17）：动态 URL 不能建立持久部署身份。
8. **Map 插入序即渲染序**（discovery.ts:30-31）："config adds global then nearest-to-farthest project files; sibling contributors interleave by transform registration order."
9. **diff 优于全文**（discovery.ts:146）：变更渲染 unified diff，但比整文短才用。
10. **合成消息是唯一台账**（session/instructions.ts:35-39，见 §10）：压缩/回退挤出后重注入而非静默丢失。
11. **引用相等判别非结构**（index.ts:383-385）："identity, never structure, discriminates"——typed source 的 A 可能恰好长得像 Unavailable 单例。
12. **空渲染即 bug**（index.ts:402-404）：`requireText` 对空 initial/changed/removed 抛错。

## 10. 失败与恢复

- **指令文件读取失败**：config/plugin/instruction.ts `isolate`（:161-168）把单源异常降级为该路 `Instructions.unavailable` 并 logWarning；已有 previous 时 diff 跳过（index.ts:297，存储值继续生效）；URL 抓取失败保留上次副本（:132-134）。首次基线被 Unavailable 阻塞时，llm.ts:204 注释："A blocked initial instruction baseline must leave admitted input pending"——inbox 输入不 promote，等待恢复。
- **超大指令**：instruction_entry 的 put 有 256KiB 硬限（`ValueTooLargeError`，entry.ts:135-141，实际字节数以 `JSON.stringify` UTF-8 计）；发现链与 API 指令源**无截断逻辑**——AGENTS.md 全文进入上下文（diff 渲染仅优化增量体积）。上下文超长的兜底是 compaction 的 overflow 路径（7_compaction 篇），不在本模块。
- **@-提及指令文件**（session/instructions.ts）：`load({sessionID, paths})`（:46-96）以 in-flight claim（`Ref<Map<ID,Set<string>>>`，:44）去重并发读取（:48-66）；`previouslyInjected`（:99-111）扫 synthetic 消息 metadata 的 `instruction.paths` 台账防止重复注入。注释（:35-39）点明失效恢复："paths whose synthetics drop out of model-visible history (compaction, revert) are re-discovered and re-injected instead of staying silently lost." 读失败文件被 `readFileStringSafe` 静默跳过（:65-68）。绕过 `Session.synthetic` 直接 publish（:70-73 注释）是为避免 Location 层依赖 Session 形成 "type cycle with this node"。
- **blob 缺失**：内容寻址下的缺 blob 是 `Effect.die`（state.ts:171/246-247；entry 侧 requireBlob 同构 :270-274）——存储损坏，无恢复路径。
- **Location 漂移**：select 发现 session 的 location 与当前不符 → `Effect.interrupt`（context.ts:124-125）——会话已在别处接管。

## 演进注记（对照旧笔记 4.17 system-context 篇，2026-08-03 基线）

| 旧笔记（4.17） | 现码（v2.0.30 实测） |
|---|---|
| `system-context/index.ts` 320 行代数核心 | 目录已删除；代数迁至 `instructions/index.ts`（256 行），Source 从"渲染函数对"升级为 key+codec+render 的类型化 `Definition` 闭包（index.ts:237-271），codec 保证 typed↔Json 双向可解码（历史 blob 重放兼容） |
| system-context/builtins | `instructions/builtins.ts`（60 行），仍是 core/date + core/environment 两源，渲染文案基本未变 |
| `context-epoch.ts` 174 行独立文件 | 已删；epoch 收缩为 `instruction_state` 表的 `epoch_start/through_seq` 两列（sql.ts:167-175）+ `advanceEpoch`（state.ts:139-154，由 Compaction.Ended 投影 projector.ts:733 驱动）+ revert 重置（:793）。4_session.md §演进.2 已考证此点，本篇补充：fork 改用 `InstructionState.current` 播种（session.ts:344） |
| Context Epoch 对象表达"上下文变了需重建" | 改由 delta 事件流表达：observe→diff→InstructionsUpdated→apply（§5 流水线），基线/增量分离渲染 |
| 三表未提及 | `instruction_entry/blob/state` 三表为新增（4_session §演进.2 提及表名），本篇 §4 首次完整给出分工与失效规则 |
| 未见 provider-context | 现码 provider-context.ts 承担 compaction 原生检查点 provenance/编解码（含 2.0.15 media 迁移 :64-82），与"Claude cache 断点"的直觉无关 |
| 指令发现层级 | 实测：全局 AGENTS.md + start→stop 祖先链 + config `instructions`（glob/绝对/URL）；`project/global` 开关（discovery Options，:47-51） |
| 旧笔记篇幅主体（320 行单文件） | 现分散为 10 文件 1,398 行——"代数"未变薄，而是拆成纯库+持久化+发现+注入四层 |

## 相关篇

- [4_session.md](4_session.md)——§6 读取路径（messageEntries/boundary）、inbox→promote 时序、projector 总览
- [9_tool.md](9_tool.md)——Tool.Snapshot 如何进 select 与 system 模板工具指导
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)——baseTranscript 之后的请求/流式/工具执行时序
- 8_agent（未写，占位）——agent.system 覆盖默认模板的规则、agent permissions 如何收窄 discovery 与 tools（context.ts:130-131）
- 7_compaction（未写，占位）——overflow 触发、native checkpoint 窗口、providerContext 存储细节
