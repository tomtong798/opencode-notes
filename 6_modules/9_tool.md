# 工具系统详解（注册 → 物化 → 执行 → 结算：opencode v2 的本地工具管线）

> 一句话：**工具 = 一个带 schema 的 `Tool.Info` 结构值，经 plugin/MCP 注入 Location 级注册表，由 `Snapshot.execute` 单口执行（解码 → 强制 Tool.Error 契约 → 编码 → 归一化），再由 Session Step 用 fiber 并行结算、截断落盘。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-19）。
> 权威出处：`specs/v2/tools.md`（语义规范）+ `packages/core/src/tool/AGENTS.md`（架构约束）
> [返回 README](README.md) | 前置 [4_session.md](4_session.md) | 关联 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md) §5（工具 fiber 并行执行时序）
> 全文 500 行，全部行号实测；重构索引见附录 B 末尾的复测指引。

---

## 0. 模块地图

实测 `wc -l`（2026-10-19，v2.0.30 dev）。工具系统横跨三个包：Schema（契约）、`@opencode/ai`（LLM 侧）、Core（注册/执行/内置）。

```
packages/
├── schema/src/tool.ts .................... 102  Tool.Info / Tool.Context / Tool.Error / Tool.Content 契约
├── ai/src/
│   ├── tool.ts ........................... 272  @opencode/ai 的 Tool.make（typed/dynamic 双模式，LLM 客户端侧）
│   ├── tool-runtime.ts ................... 111  ToolRuntime.dispatch：@opencode/ai 自有工具结算（非 Core 路径）
│   └── tool-history.ts ................... 98   工具调用历史（本篇不展开）
└── core/src/
    ├── tool.ts ........................... 332  Tool.Service：Location 级注册表 + Snapshot（执行单口）
    ├── tool-output.ts .................... 150  ToolOutput.Service：输出截断 + 全量落盘（7 天保留）
    ├── tool/
    │   ├── AGENTS.md ..................... ~150 架构约束（非代码）
    │   ├── runtime.ts .................... 281  definition()/execute()：解码、契约强制、归一化、JSON Schema 转换
    │   ├── read-filesystem.ts ............ 402  read 工具的文件分页引擎（2000 行/50KiB 上限）
    │   ├── mcp.ts ........................ 152  McpTool：MCP 工具桥接为内部 Tool.Info
    │   ├── html-markdown.ts .............. 658  webfetch 的 HTML→Markdown 转换
    │   ├── http-body.ts .................. 30   webfetch 的响应体收集
    │   ├── patch.txt ..................... 设计残稿（非源码，忽略）
    │   └── plugin/ ....................... 内置工具（15 文件 / 16 个工具名）
    │       ├── read.ts ................... 211  read
    │       ├── write.ts .................. 101  write
    │       ├── edit.ts ................... 332  edit
    │       ├── patch.ts .................. 363  patch
    │       ├── glob.ts ................... 139  glob
    │       ├── grep.ts ................... 170  grep
    │       ├── shell.ts .................. 293  shell（旧笔记的 bash，已改名）
    │       ├── question.ts ............... 132  question
    │       ├── skill.ts .................. 72   skill
    │       ├── subagent.ts ............... 311  subagent（旧笔记的 task；todowrite 已删）
    │       ├── webfetch.ts ............... 189  webfetch
    │       ├── websearch.ts ............... 196  websearch
    │       ├── mcp-resource.ts ........... 101  list_mcp_resources / read_mcp_resource（2 个工具）
    │       ├── opencode.ts ............... 299  opencode 命名空间：session_rename/session_move/worktree_*/models 等
    │       └── file-diff.ts .............. 23   权限预览用的 diff 生成（write/edit/patch 共用）
    └── mcp/
        ├── index.ts ...................... 846  Mcp.Service：服务器连接/工具发现/callTool
        ├── client.ts ..................... 446  MCP 客户端传输
        ├── stdio.ts ...................... 196  stdio 传输
        ├── oauth.ts ...................... 532  远程 MCP 的 OAuth 集成
        └── instructions.ts ............... 111  MCP 提示注入
```

注册入口在 `core/src/plugin/internal.ts:242-250`：`ReadTool.Plugin`、`ShellTool.Plugin`、`EditTool.Plugin`、`GlobTool.Plugin`、`GrepTool.Plugin`、`OpenCodeTools.Plugin`、`McpResourceTools.Plugin`、`QuestionTool.Plugin`、`SkillTool.Plugin`、`SubagentTool.Plugin`、`WebFetchTool.Plugin`、`WebSearchTool.Plugin`、`WriteTool.Plugin`、`PatchTool.Plugin` 以**内部 plugin** 身份注入——内置工具与第三方 plugin 走同一条 `ctx.tool.transform(editor => editor.add(...))` 注册路径，没有任何特权通道。

## 1. 心智模型

四段生命周期：

```
[① 注册]  plugin (内置 15 个 + 用户 plugin) ──ctx.tool.transform──▶ Tool.Service (Location 级)
           MCP 发现 ──tools.transform(editor.add)──▶ 同一注册表 (tool/mcp.ts:41)
                │
                │  State<Editor>：同步 transform 按注册序重放；effectiveName = <ns>_<name>
                ▼
[② 物化]  Tool.Service.snapshot(permissions)  (tool.ts:225)
           ├─ whollyDisabled 过滤（catalog 可见性，非执行授权）
           ├─ codemode:false → direct 工具（直接出现在 provider tool 列表）
           ├─ 其余 → CodeMode inventory，折叠为一个 "execute" 工具
           └─ 产出 Snapshot { definitions[], codeModeCatalog, execute }
                │
                ▼
[③ 执行]  Snapshot.execute({sessionID, agent, messageID, call, progress})  (tool.ts:263)
           ├─ execute.before hook（plugin 可改写 input）        (tool.ts:103)
           ├─ runtime.execute：解码 → 契约强制 → tool.execute → 编码输出
           │   └─ 工具内部：FileAccess 解析 → permission.assert → 副作用
           ├─ execute.after hook                              (tool.ts:149)
           └─ normalizeContent + normalizeImages 归一化
                │
                ▼
[④ 结算]  Session runner (session/runner/step.ts:122-134)
           ├─ 每个非 providerExecuted 的 tool-call fork 一个 fiber
           ├─ ToolOutput.truncate（2000 行/50KiB，超限写盘）  (tool-output.ts:67)
           └─ publisher.toolExecution / failTool → durable 落库
```

关键点：注册表本身**无权限依赖**（`tool/AGENTS.md`："The registry has no `Permission.Service` dependency"）；权限 assert 发生在每个工具的 execute 内部（叶节点），注册表只做 `whollyDisabled` 的目录过滤（`tool.ts:292-295`）。

## 2. 定义接口

### 2.1 Core 侧：`Tool.Info`（schema/tool.ts:92-102）

现码没有 specs 里的 `Tool.make`——Core 用的是**透明结构值**：

```ts
// packages/schema/src/tool.ts:92-102
export type Info<Input, Output> = {
  readonly name: string
  readonly input: Input
  readonly description: string
  readonly execute: (input, context: Context) => Effect.Effect<Result<Output>, Error>
  readonly output?: Output
  readonly options?: Options
}
```

- **Context**（tool.ts:14-20）：`sessionID / agent / messageID / id: CallID / progress`。与 specs 一致：runner 拥有 message 关联，registry 不推断。
- **Result**（tool.ts:86-90）：`{ output?, content?, metadata? }` 三值分离——`output` 是 CodeMode 机器值（ephemeral），`content` 是模型可见值（durable），`metadata` 给 UI。
- **Error**（tool.ts:61-65）：`Tool.Error` tagged error，可选携带 `error: Defect` 与 metadata。
- **Options**（tool.ts:32-42）：`namespace?`（有效名 `<ns>_<name>`）、`permission?`（动作别名，edit/write/patch 共用 `"edit"`）、`codemode`（默认 true 入 CodeMode；`false` 直挂 provider 列表；`codemode:false` 时名字 `execute` 保留给 CodeMode，tool.ts:313-314）。
- **ValueSchema**（tool.ts:44）：`Schema.Codec | StandardSchemaV1 | JsonSchema` 三选一——**Effect Schema、zod（v4，走 Standard Schema）、裸 JSON Schema 都能注册**。

#### Context 完整字段（schema/tool.ts:14-20）

```ts
export interface Context {
  readonly sessionID: Session.ID
  readonly agent: Agent.ID
  readonly messageID: SessionMessage.ID   // 含本次调用的助手消息 durable ID
  readonly id: CallID                     // 与 durable 事件同源的调用标识
  readonly progress: (update: Metadata) => Effect.Effect<void>
}
```

specs 的"runner 拥有 message 关联、registry 不推断"在 core/tool.ts:264-270 落地：`Snapshot.execute` 的调用方（model-request → step）必须传入完整四元组，`progress` 缺省为 no-op。原始 provider 入参与领域服务都不得进 Context（specs："Raw provider input and domain services do not belong in the invocation context"）。

#### Content 模型（schema/tool.ts:67-84）

模型可见内容只有两种原子：`TextContent { type:"text", text }` 与 `FileContent { type:"file", uri, mime, name? }`。`normalizeImages`（core/tool.ts:74-101）只处理 data-URI 的 `image/*`：经 `Image.normalize` 重采样（解码失败/超限则换成 `[N images omitted: ...]` 注记文本）；非图片 file 项原样透传。

### 2.2 runtime.ts 的双支持验证

- `runtime.ts:135-136`：`isStandardSchema` 检查 `"~standard" in schema`（zod v4 实现 Standard Schema V1）。
- `runtime.ts:168-172`：`standardJsonSchema` 里 `schema instanceof $ZodType` 走 `toJSONSchema(schema, { io })`（`zod/v4/core` 导入，runtime.ts:6）——zod schema 转模型侧 JSON Schema。
- `runtime.ts:71-86` `validateInput`：Standard Schema 走 `validate()`；Effect Schema 走 `Schema.decodeUnknownEffect`；裸 JsonSchema 经 `jsonSchemas` Cache（runtime.ts:10-19，capacity 100）转成 Codec 再解码。
- `runtime.ts:102-108`：JSON Schema 按 draft-07 / 2020-12 自动判别转 Effect Codec。

### 2.3 `@opencode/ai` 侧：`Tool.make`（ai/tool.ts:135-208）

`@opencode/ai` 的 `Tool.make` 是**另一套**、面向 LLM 客户端的工具构造器：typed（Effect `parameters`/`success` schema，自动 decode/encode）与 dynamic（裸 `jsonSchema`，MCP/manifest 场景）双模式（ai/tool.ts:104-134 文档注释）。内部 memoize `_decode/_encode/_project/_definition`（ai/tool.ts:57-70），`toDefinitions` 用 record key 重建带名字的 `ToolDefinition`（ai/tool.ts:223-232）。Core 的 CoreMode 折叠工具最终也以 `ToolDefinition`（来自 `@opencode/ai`）形态进入 `LLM.stream` 请求。

`ai/tool-runtime.ts:23-42` 的 `dispatch` 是 `@opencode/ai` 包自带的"无 Session"结算路径（decode → execute → encode → project 成 ToolOutput + LLMEvent），与 Core 的 runtime.ts 同构但**不含权限/hook/落盘**——Core 不用它，Core 用 `tool/runtime.ts`。

### 2.4 两个 JSON Schema 兼容补丁

`ai/tool.ts:243-248` 的 `emptyInputJsonSchema`：Effect 会为"无属性 struct"生成 `{ not: { type: "null" } }`，但 provider 要求参数必须是 object——该函数把它改写为 `{ type:"object", properties:{}, additionalProperties:false }`，仅当剩余关键字都是注解类（title/description/default 等，:250）时才动手。Core 的 runtime.ts:160 在派生 inputSchema 时同样调用它。

`runtime.ts:176-187` 的 `toJsonSchema` 归一化（flattenAllOf + inlineLocalReferences）见 §3.1。

## 3. 执行管线

### 3.0 全链路时序（单次工具调用）

```
provider stream 事件 tool-call (providerExecuted=false)
  │ session/runner/step.ts:122
  ▼
fork fiber (uninterruptibleMask + restore)          step.ts:125
  │
  ├─ SessionModelRequest.executeTool                model-request.ts:404
  │    └─ tools.execute({...call, definitions: hooked})
  │         │ core/tool.ts:263 Snapshot.execute
  │         ├─ 组 Tool.Context (CallID.make(call.id))  tool.ts:264-270
  │         ├─ hooks.trigger("tool","execute.before") tool.ts:103-111 ← plugin 可改 input
  │         ├─ definitions 裁决（hook 删除→报不可用）   tool.ts:272-283
  │         └─ executeTool(tool, name, event.input, context)  tool.ts:113
  │              ├─ runtime.execute                  runtime.ts:29
  │              │    ├─ decodeInput                runtime.ts:63
  │              │    ├─ 强转 foreign failure        runtime.ts:37-45
  │              │    ├─ tool.execute(decoded, ctx)  ← 权限 assert + 副作用都在这里
  │              │    └─ encodeOutput                runtime.ts:110
  │              ├─ hooks.trigger("tool","execute.after") tool.ts:140-149
  │              └─ normalizeContent + normalizeImages    tool.ts:150
  │
  ├─ toolOutput.truncate                             step.ts:127 / tool-output.ts:67
  └─ publisher.toolExecution → durable 事件          step.ts:128
```

### 3.1 definition()：注册表 → 模型侧定义

`runtime.ts:21-27`：`effectiveName(tool)`（namespace 展平 + 非法字符换 `_`，runtime.ts:276-281）+ description + inputJsonSchema + 可选 outputJsonSchema。注册时 `registrationError`（core/tool.ts:304-321）会实际构造 `ToolDefinition.make` 验证定义合法，`execute` 名保留给 CodeMode。

`toJsonSchema`（runtime.ts:174-188）做了两件供应商兼容工程：

- `flattenAllOf`（runtime.ts:190-199）：把 `{type:"integer", allOf:[{minimum:0}]}` 压平成 `{type:"integer", minimum:0}`，只在没有任何关键字会被覆盖时执行（`canFlattenAllOf`，:204-213）。
- `inlineLocalReferences`（runtime.ts:215-243）：把 `$defs` 的本地 `$ref` 内联展开（`canMergeRecords` 无键冲突直接合并，否则退回 allOf）；递归引用保留原样——注释明说"expanding them would never terminate"；最后 `dropDefinitionsIfResolved`（:248-252）清掉不再被引用的 `$defs`。

这两段都源于同一句注释（runtime.ts:176-177）："Effect emits valid JSON Schema that some inference providers handle poorly."——即合法 JSON Schema ≠ 所有推理供应商都能消化。

### 3.2 execute()：四步契约（runtime.ts:29-61）

```
execute(tool, input, context)
  ├─ ① decodeInput（runtime.ts:63-69）：失败 → Tool.Error，错误消息含最多 5 条 issue
  │     + 原始入参 JSON + "Update the arguments and call the tool again"（runtime.ts:88-100）
  ├─ ② tool.execute(decoded, context) 但包一层 mapError（runtime.ts:37-45）：
  │     // 原文注释（runtime.ts:32-36）：
  │     // "Tool implementations declare `Tool.Error` but plugins can fail with anything at
  │     //  runtime. A foreign typed failure would slip past every `catchTag("Tool.Error")`
  │     //  downstream and leave its call permanently unsettled, so the declared contract is
  │     //  enforced here at the untrusted boundary. Declines tunnel through as defects and
  │     //  interrupts are not errors; neither is touched."
  │     → 非 Tool.Error 的 typed failure 一律强转 Tool.Error（foreign typed failure 防御）
  ├─ ③ 输出编码（runtime.ts:46-55）：
  │     无 output schema：result 带 output → Effect.die（声明矛盾）
  │     有 output schema：result 缺 output → Tool.Error；否则 encodeOutput（runtime.ts:110-133）
  └─ ④ normalizeContent（runtime.ts:261-265）：string → 单 text 项；空数组/undefined → stringify(output)
```

然后 core/tool.ts 的 `executeTool`（tool.ts:113-156）接手：`execute.before` hook（可修复 input，tool.ts:103-111）→ runtime.execute → `execute.after` hook（可改写 outcome，tool.ts:140-149）→ `normalizeContent` 重归一化 → `normalizeImages`（data-URI 图片过 Image.normalize 重采样，失败换省略注记，tool.ts:74-101）。

### 3.3 Snapshot.execute：唯一执行口（tool.ts:263-284）

从 `input.call` 组 `Tool.Context`（CallID.make），before hook 后按 `input.definitions`（本请求实际广告的定义表）做可用性裁决：hook 删掉的工具报 `Tool is not available for this request`；`execute` 名转给 CodeMode 折叠工具；direct 表命中即执行；否则报 `No tool named "..." is currently available`。**"捕获执行"不变量**：每个模型请求捕获当时的定义与执行器，之后注册变化只影响后续请求（AGENTS.md "Captured execution"）。

### 3.4 snapshot()：物化细节（tool.ts:225-287）

```
snapshot(permissions?)
  ├─ 遍历注册 tools（Map<effectiveName, Tool.Info>）
  │    └─ whollyDisabled(options.permission ?? name, rules) → 跳过      tool.ts:230-233
  ├─ direct     = codemode === false 的工具                            tool.ts:234
  ├─ codeModeTools = 其余（进 CodeMode 目录）                          tool.ts:235
  ├─ CodeModeTool.create(inventory, executor)（"execute" 折叠工具）     tool.ts:239-245
  │    └─ codeModeEnabled 由 whollyDisabled("execute") 决定            tool.ts:238
  ├─ catalog 缓存：data 指针 + 名单串一致则复用（跨 step 共享大目录）    tool.ts:246-254
  └─ definitions = direct 排序后的 definition() + [codeModeTool]        tool.ts:257-262
```

注意 catalog 缓存的注释（tool.ts:247-248）："Discovery is immutable for a registry revision ... share the much larger rendered catalog across steps"——目录渲染昂贵，按（数据对象身份 + 名单）双键缓存。

### 3.5 注册状态机：State + Editor（tool.ts:159-219）

`State.create` 是 Core 通用的"同步 transform 重放"容器：每次 `transform(fn)` 把 fn 记入序列，`reload()` 用捕获的源数据对全新 editor 依序重放。Editor 操作语义（tool.ts:166-205）：

- `add`：先 `registrationError` 校验（非法名字/非法 namespace/定义不可构造/占用 `execute`），失败进 `errors[]` 并记日志跳过，**不抛异常**；成功以 effectiveName 为键存入（后注册覆盖先注册——"latest wins"）。
- `update(id, fn)`：名字与 namespace 不可改（强制回填，:191-194）；非法更新保留旧定义。
- `remove(id)`：仅删该键，揭示更早的被覆盖注册。
- `notify`：每次重放后清 catalog 缓存并输出错误日志（:206-218）。

## 4. 内置工具走查

15 个文件 / 16+ 个工具名（opencode.ts 与 mcp-resource.ts 各注册多个）。汇总表（行数实测）：

| 工具 | 文件 | 行数 | 权限 action | resources | codemode | 输出形态 |
|---|---|---|---|---|---|---|
| read | plugin/read.ts | 211 | （读授权走 FileAccess） | — | false | FileContent/TextPage/ListPage + 行号文本 |
| write | plugin/write.ts | 101 | `edit` | [target.resource] | false | 写入结果 + fileDiff 预览 |
| edit | plugin/edit.ts | 332 | `edit` | [target.resource] | false | 替换结果 + diff |
| patch | plugin/patch.ts | 363 | `edit` | [去重后的 targets] | false | 逐 hunk 应用结果 |
| glob | plugin/glob.ts | 139 | `glob` | [pattern] | false | 匹配路径列表 |
| grep | plugin/grep.ts | 170 | `grep` | [pattern] | false | 匹配行 |
| shell | plugin/shell.ts | 293 | `shell` | 解析出的每条命令 | false | 输出 + 可选后台/stream 文件 |
| question | plugin/question.ts | 132 | `question` | ["*"] | false | answers 表单结果 |
| skill | plugin/skill.ts | 72 | `skill` | [skill.id] | false | skill 注入文本 + metadata |
| subagent | plugin/subagent.ts | 311 | `subagent` | [agent.id] | false | 子会话摘要 |
| webfetch | plugin/webfetch.ts | 189 | `webfetch` | [url] | false | markdown/text 内容 |
| websearch | plugin/websearch.ts | 196 | `websearch` | [query] | false | 搜索结果 |
| list/read_mcp_resource | plugin/mcp-resource.ts | 101 | `opencode_list/read_mcp_resource` | servers / `server:uri` | true | 资源目录 / 资源内容 |
| opencode::* | plugin/opencode.ts | 299 | （session_rename 等无 assert；worktree_remove 有） | — | true | 管理操作结果 |

### 4.1 read（read.ts:29-173）

- 输入：`path + offset + limit`（1-based，默认与上限 2000 行，read.ts:17-25）。
- 服务：`ReadToolFileSystem.Service`（分页引擎，read-filesystem.ts：`MAX_READ_LINES=2000`、`MAX_READ_BYTES=50KiB`、行超 2000 字符截断，:14-18）。
- 特色路径：`access.authorizeRead`（:55）→ 读失败时 NFC/引号规范化"备选路径"重试（`alternatePath`，:138-150）→ 仍失败给"Did you mean"建议（`missing`，:152-171）→ 成功后向上走查 AGENTS.md 注入为 durable 指令（:82-104，失败不影响读）。
- 输出：`toModelContent`（:175-211）文本页带 `<n>: <content>` 行号前缀，截断时附 `Continue reading with offset: N`；图片/PDF 直接以 FileContent data-URI 给模型。

### 4.2 write（write.ts:57-101）

`access.resolve` → `authorizeExternal` → 读旧文本 → `fileDiff` 预览 → `permission.assert({action:"edit", save:["*"], metadata:{files:[preview]}})`（:78-85）→ `writeTextPreservingBom` 写入（BOM 保持）→ formatter 若配置则同步。options 显式 `permission: "edit"`（:60），使目录过滤按 edit 动作。

### 4.3 edit（edit.ts:215-300+）

精确替换。权限 assert **先于**替换计数检查（:275-283）：assert 通过后若 `replacements===0` / 多匹配未开 replaceAll / 替换后无变化，均报 ToolFailure 给模型自纠（:284-300）。预览 diff 同样进 metadata。

### 4.4 patch（patch.ts:79-160+）

多文件 add/update/delete 补丁。先 `Patch.parse` 静态验证，锁目标集合；逐 hunk 准备后一次性 `permission.assert({action:"edit", resources:[...new Set(targets)]})`（:196-198）；失败时报"Completed before failure: ..."部分进度（:92-97）。

### 4.5 glob / grep（glob.ts:60-95；grep.ts:80-125）

两者同构：resolve → authorizeExternal → assert（resources=[pattern]，metadata 携带搜索参数）→ ripgrep 执行（limit+1 探测截断）→ 超时 `DEFAULT_SEARCH_TIMEOUT_MS` 报错。path 缺省 `.`。

### 4.6 shell（shell.ts:193-293）

旧笔记的 `bash` 已改名 `shell`。命令经 `ShellParse.scan` 解析出子命令与涉及目录（prepare，:117-145），权限按**每条解析命令** assert（resources/save 都是逐命令，:138-144）——即 `a && b` 两条命令分别授权。执行走 `shell.create`（env 注入 `OPENCODE_SESSION_ID` 等）→ `jobs.start`（含 recovery 记录）→ 前台等结果或 `background:true` 立即返回 `Output is streaming to: <file>`（:96-99）。输出超限由 Shell 层存 backing 文件、返回截断尾 + 全量路径（AGENTS.md"Shell stores combined process output in its backing file"）。producer 自限（`truncated` metadata 为 true 时 ToolOutput.truncate 跳过，见 §7）。

执行内部结构（:200-293）：

```
execute
  ├─ timeout：background ? (input.timeout ?? 0) : (input.timeout ?? DEFAULT_TIMEOUT_MS)
  ├─ shell.create({command, cwd, timeout, shell}, before)   before: env 注入 + prepare()
  │     └─ prepare：resolve cwd → ShellParse.scan → authorizeExternal([target, ...dirs])
  │                  → permission.assert(逐命令)              shell.ts:117-145
  ├─ context.progress({shellID})                            即时上报 shell 句柄
  ├─ jobs.start({ id: info.id, recovery: {kind:"shell",...}, run })   挂进 job 系统
  │     └─ run：shell.result → ShellResult.output → 超时缀注 → Deferred.succeed(settled)
  │              + Effect.onInterrupt(() => shell.remove)    中断即清理
  ├─ background === true → jobs.background + notifyWhenDone(完成时通知) → 立即返回流文件路径
  └─ 否则等待 job 完成 → resultMessages(output).join("\n\n")
```

description（:27, :40-43）本身就是行为契约：后台命令"DO NOT poll for completion"、大输出"full result is saved to a file and a truncated preview is returned"——模型侧的期待由工具描述与 §7 的实现共同兑现。

### 4.7 question（question.ts:55-100）

先 `permission.assert({action:"question", resources:["*"]})`（无 save），再 `forms.ask` 表单。**取消语义双分支**（:90-95 注释）：带 message 的取消是"模型可读反馈"（ToolFailure 继续对话）；纯取消 `Effect.die(new CancelledError())`——"Deliberate defect tunnel ... it never becomes model-facing tool output"，在 `SessionModelRequest.executeTool` 处复现为 typed failure（见 §6）。

### 4.8 skill（skill.ts:40-72）

`skills.get(id)` → assert（resources/save=[skill.id]）→ `Skill.prepare` 展开指令文本。输出 content 即注入文本，metadata 带 name/directory。

### 4.9 subagent（subagent.ts:102-175+）

递归深度守卫（向上数 parentID 链，超 `experimental.subagent_depth` 默认 1 报错，:114-131）→ agent 解析（primary agent 禁止，:133-136）→ assert（resources/save=[agent.id]）→ 支持续接既有子会话（sessionID 校验父子关系 + 换 agent/model）。agent 定义里的权限经 `Permission.evaluate(...).effect !== "deny"` 预筛（:293）。

### 4.10 webfetch（webfetch.ts:108-155+）

URL 校验 → assert（resources=[url]）→ 抓取（Cloudflare 挑战自动用 opencode 代理重试，:131-133）→ 非 textual mime 拒绝 → `html-markdown.ts` 转换 → 超时默认值控制。

### 4.11 websearch（websearch.ts:42-100+）

assert（resources=[query]）→ 查询；无默认 provider 时经 `providerSelectionLock` 串行化弹表单让用户选（:60-75），进度经 `context.progress({provider})` 上报。

### 4.12 mcp-resource（mcp-resource.ts:17-101）

两个工具：`list_mcp_resources`（action=`opencode_list_mcp_resources`，resources=servers，:36-46）与 `read_mcp_resource`（action=`opencode_read_mcp_resource`，resources=[`server:uri`]，:65-80）。description 明说"oversized output is truncated automatically and the full content is saved to a file"——即依赖 §7 的通用有界化。

### 4.13 opencode 命名空间（opencode.ts:100-299）

注册 `opencode` namespace 描述（:107-110）后添加：`session_rename`（:111）、`session_move`（:131）、`worktree_create/list/remove`（:156/:170/:182）、`models`（:220）、`opencode`（主入口，:106）。均 `codemode: true`（进 CodeMode 目录）。多数无 permission.assert（会话元数据操作），`worktree_remove` 的强制删除路径有 edit 断言（:192-194）。这一族工具还演示了第三种输入模式：`session_rename` 直接返回 `{ output, content }` 二元组（:121-125），无 metadata、无文件副作用——是最小的合法 Result 形态。

### 4.14 file-diff.ts（fileDiff，23 行）

write/edit/patch 共用的统一 diff 预览生成器（`file-diff.ts:1-23`），产出进 `permission.assert` 的 `metadata.files`——即权限审批 UI 展示的 diff 就是它生成的。

## 5. MCP 桥接

```
Mcp.Service (mcp/index.ts:846 行)
  ├─ 连接管理：stdio / remote(oauth) / 配置热更
  ├─ tools() → Mcp.Tool[] (= McpClient.Tool + server + codemode, index.ts:36)
  └─ callTool({server, name, args, sessionID})  (index.ts:669)
          ▲
          │ 每个 MCP 工具包成一个内部 Tool.Info
McpTool.layer (tool/mcp.ts:26-146)
  ├─ 初始发现后一次性 tools.transform(editor.add)（:37-123）
  ├─ 名字：namespace(server) + "_" + 工具名，非法字符换 _（:16-17）
  ├─ input/output 直接用服务器 JSON Schema（缺省 {} 兜底，:47-48）
  └─ execute（:49-117）：
      ① permission.assert({action: <server>_<tool>, resources:["*"], save:["*"],
            metadata:{input}, source:{type:"tool", messageID, id}})   ← MCP 工具统一前置授权
      ② mcp.callTool → MCP.NotFoundError/ToolCallError → ToolFailure（:72-77）
      ③ result.isError → ToolFailure（文本拼接，:78-85）
      ④ content 映射：text→text 项；二进制→data-URI file 项（:86-94）
      ⑤ output：structured 优先；无 schema 时尝试把 "{..}" 文本 JSON.parse（:96-106）
      ⑥ 末尾 mapError 兜底强转 ToolFailure（:112-116）
```

工具目录变化：`McpEvent.ToolsChanged` → sliding PubSub → 100ms debounce → `reconcile`（重新 `mcp.tools()` + `tools.reload()`，:124-143）。关键设计：**MCP 只持有一个稳定 transform，刷新走 reload 而非重新注册**，因此保住了后注册 plugin 覆盖 MCP 工具的优先级（AGENTS.md"MCP owns one stable tool transform"）。debounce 的理由见 :131-132 注释："Servers announce tools in bursts and each read loads the whole catalog, so settle and refresh once."

MCP 侧 `Tool` 类型 = `McpClient.Tool & { server, codemode? }`（mcp/index.ts:36）；`Interface.tools()` 聚合所有已连接服务器的工具（index.ts:120-122），`callTool` 在 index.ts:669。mcp/ 目录其余文件（client/stdio/oauth/instructions）属于连接与传输层，本篇不展开。

## 6. 权限接缝

详细规则归 [11_permission.md]（未来篇），本篇只写工具侧三点：

1. **assert 时机——叶子**：每个内置工具在自己的 `execute` 内、副作用之前 assert（§4 各节行号）。注册表不做执行授权；`options.permission` 只影响 `whollyDisabled` 目录过滤（core/tool.ts:229-232, 292-295）。
2. **decline 隧道**：`Permission.assert` 把用户拒绝包成 `Effect.die(DeclinedError)` 故意穿透叶子的 `mapError` 毯子（permission.ts:248-253 原注释 "Deliberate defect tunnel: leaves wrap execution in blanket `mapError`, which ... a 'no' must never become model-visible output"）；带反馈的拒绝（`CorrectedError`）保持 typed，叶子转成 ToolFailure 让模型继续。
3. **复现点**：`SessionModelRequest.executeTool`（session/model-request.ts:404-421）用 `catchCauseFilter` 从 cause 里捞出 `Permission.DeclinedError | QuestionTool.CancelledError` 复现为该调用的 typed failure；Step 层再把它分类为 decline → 该工具以 "The user declined this tool call" 失败、整个 Step 中断（step.ts:207-215, 274-277）。

runtime.ts:35 的 "Declines tunnel through as defects" 与 question.ts:90-95、model-request.ts:404 三处注释互相印证，构成同一条隧道。完整链路：

```
用户点"拒绝" (Permission UI)
  → Deferred.fail(DeclinedError)                     permission.ts:118
  → Permission.assert 出口 catchTag → Effect.die(DeclinedError)   permission.ts:253   [进 defect 通道]
  → 穿透叶子工具的 mapError 毯子（AGENTS.md 禁 catchCause）
  → 穿透 runtime.ts:37 的 foreign-failure 强转（只动 error 通道，runtime.ts:36）
  → SessionModelRequest.executeTool catchCauseFilter     model-request.ts:406-418  [捞回]
  → Step classifyToolExits 归为 decline                  step.ts:303-308
  → failTool("The user declined this tool call") + Step 中断   step.ts:207-215, 274-277
```

## 7. 输出有界化

**演进**：旧笔记的 `tool-output-store.ts` 已不存在（全仓 grep 无命中）；现为 `core/src/tool-output.ts`（150 行），职责收窄为"截断 + 落盘"。

```
SessionStep（runner/step.ts:126-128）
  restore(executeTool(event))          ← Snapshot.execute（含 hook/归一化）
    → Effect.flatMap(toolOutput.truncate)   ← 通用有界化在此
    → publisher.toolExecution(...)          ← durable 发布
```

`ToolOutput.truncate`（tool-output.ts:67-118）：

- `metadata.truncated !== undefined` → 原样返回（**producer 自限优先**：shell、read 等已自带截断语义的工具不二次截断）。
- 只测 text 项；默认上限 `MAX_LINES=2000`、`MAX_BYTES=50KiB`（:13-14，可经 `ConfigToolOutputPlugin` 的 transform.configure 调整）。
- 超限：逐行累计到字节上限 → 全文写 `<global.data>/tool-output/tool_<id>`（:86-88）→ 内容替换为保留前缀 + `[showing lines 1-N of M; full output saved to <file>]` 标记（:90-112）→ metadata 记 `truncated: true, outputPath`。
- 后台清理：每小时删 7 天前的文件（`RETENTION=Duration.days(7)`，:15, 129-138）。

三层限界各司其职（对应 specs "Producers And The Registry Own Different Limits"）：

| 层 | 落点 | 上限 | 失真责任 |
|---|---|---|---|
| producer 自限 | read（2000 行/50KiB 分页）、shell（backing 文件+截断尾）、glob/grep（limit+1 探测） | 各自定义 | 必须如实报告自己的丢失（`metadata.truncated`） |
| 通用有界化 | `ToolOutput.truncate` | 2000 行/50KiB（可配置） | 只处理 text 项；file 项不动 |
| 压缩摘要 | compaction.ts:813 `truncateToolOutput` | 独立策略 | 历史压缩时的二次截断，不影响已存内容 |

对比 specs 的差异注记：specs 描述"head-plus-tail split"（头尾保留），现码是**头保留 + 落盘指针**——规范先行、实现简化，属可观察漂移。specs 的"if complete retention fails, execution fails operationally"对应 `writeFileString(...).pipe(Effect.orDie)`（:88）。

## 8. 不变量与设计取舍

对照 specs/v2/tools.md 的 Laws，逐条落点：

- **单执行**：`Tool.Info` 只有一个 `execute`（schema/tool.ts:99）；AGENTS.md 禁止第二个可执行入口类型。
- **编解码边界**：工具只见解码后输入（runtime.ts:31）；CodeMode 只见编码后 output（:55）；模型内容/metadata 来自 Result 三值分离。
- **捕获执行**：`Snapshot.execute` 用请求捕获的 definitions 裁决（core/tool.ts:272-280）。
- **durable 先于副作用 / 每个本地调用先持久化再副作用**：不是工具层直接实现，而是 Session 结构保证——Step 里"本地执行只在其 Tool.Called 出版完成后才开始"（step.ts:103-104 注释 "A local execution starts only after its Tool.Called publication completes"），工具结果经 publisher 进 durable 事件流（`publisher.toolExecution`），文件快照在 Step 收尾 diff（step.ts:236-244）。权限 assert 也一律先于磁盘/网络副作用（§4 各工具 assert → 副作用顺序）。
- **Scoped registration**：`State.create` 按注册序重放同步 transform；`update/remove` 按 effectiveName 定向；MCP 用单一 transform + reload 保持覆盖序（§5）。
- **类型安全止于注册**：注册表擦除为 `Tool.Info<any>`，运行时校验输入/输出（tool/AGENTS.md"Type safety ends at registration"）——这是 AGENTS.md 明确允许 `any` 的唯一内部边界。
- **registry 无权限依赖**：授权是叶子职责（§6）。
- **CodeMode 折叠**：默认工具进 CodeMode 目录（省 provider tool 数量、保 prompt 前缀缓存），`codemode:false` 的工具直挂；`execute` 名保留（core/tool.ts:313-314）。specs 里 "final Step retains tool definitions with toolChoice none" 对应 step.ts:92-93。

## 9. 失败与恢复

- **Tool.Error**：唯一合法的模型可见失败。runtime.ts:37-45 在不可信边界强制——任何 foreign typed failure 被强转（§3.2 引原文注释），保证下游每个 `catchTag("Tool.Error")` 都能结算，调用不会"永久 unsettled"。
- **中断不打扰**：interrupt 不是错误——强转逻辑只动 `mapError` 的 error 通道，interrupt/defect 原样穿透（runtime.ts:36 "interrupts are not errors; neither is touched"；AGENTS.md 禁 `catchCause`）。
- **Step 层结算分类**（step.ts:297-321 `classifyToolExits`）：每个工具 fiber 的 exit 分为 decline（typed: DeclinedError/CancelledError）、interrupt、die-failure 三类；decline → failTool("user declined") + Step 中断（:207-225）；中断 → `failUnsettledTools(TOOLS_INTERRUPTED)`（:224）；provider 流结束但有未结算调用 → `RESULT_MISSING`（"hosted"）（:228-231）。注释（:297）："Tool.Error settles in each fiber; only user declines remain in the typed error channel."
- **输入无效不执行**（specs Law）：decodeInput 失败在 `tool.execute` 之前（runtime.ts:31）；输出无效不成功（:54）。
- **部分失败进度**：patch 的 "Completed before failure" 模式（patch.ts:92-97）。
- **无中断结算**：step.ts:140 注释 "Keep the final tool and Step events uninterruptible, even when the work itself is cancelled"——`Effect.uninterruptibleMask` 包住 fiber join 与出版段，保证每个已开始的调用必有终态事件；流被中断时先 `interruptTools`（step.ts:90, 150-152）再统一补 `TOOLS_INTERRUPTED`。
- **重启恢复**：shell 的 `jobs.start` 携带 `recovery: { kind:"shell", sessionID, shellID, command }`（shell.ts:253-258），配合 Session 层的执行声明机制在进程重启后续接后台命令（详见 4_session 篇）。

## 演进注记

对照旧笔记 4.5（2026-08-03 基线），逐项实测：

| 旧笔记断言 | 现码（v2.0.30） | 结论 |
|---|---|---|
| `tool/tool.ts`（Tool.make，162 行）opaque 工具值 | 已删；Core 用 `schema/tool.ts` 的透明 `Tool.Info` + 各 plugin `editor.add` | **漂移**：opaque→结构值，`Tool.make` 只剩 `@opencode/ai` 侧 |
| `tool/registry.ts`（ToolRegistry，147 行）+ `materialize`/`settleWith` | 已删；合并为 `core/src/tool.ts` 的 `Tool.Service` + `Snapshot`（execute 单口） | **漂移**：物化/结算改名 snapshot/execute，职责并入单一服务 |
| `tools.ts`（Tools.Service）/`application-tools.ts` | 已删；注册走 `State.Transformable<Editor>`（`ctx.tool.transform`），内置也是 plugin | **漂移**：三层作用域 → plugin 注入单一通道 |
| `builtins.ts`（12 内置） | `plugin/` 目录 15 文件，经 `plugin/internal.ts:242-250` 以内部 plugin 注册 | **漂移**：builtin→plugin 模式确认 |
| `bash` 工具 | `plugin/shell.ts`，`export const name = "shell"`（shell.ts:23） | **确认改名** |
| `todowrite` | 全仓无此工具（grep 无命中；任务清单由宿主 TUI/plugin 层实现） | **确认删除** |
| `task`（子代理） | `plugin/subagent.ts`，name="subagent" | **改名 task→subagent** |
| `tool-output-store.ts` | 无此文件；`tool-output.ts`（truncate+落盘，7 天保留） | **漂移**：模块更名、职责收窄 |
| registry 无权限依赖 | 仍成立（tool/AGENTS.md + core/tool.ts:292 只有 whollyDisabled） | **稳定** |
| `ToolRegistry.Materialization.settle` 是唯一执行/有界化边界 | 拆成两处：`Snapshot.execute`（执行）+ Step 层 `ToolOutput.truncate`（有界化） | **漂移**：有界化上移到 runner |

另一条主线漂移：**CodeMode**（把多数工具折叠进单个 `execute` 工具的目录机制）是旧基线之后引入的，`options.codemode`、`CodeModeCatalog`、`execute` 名保留都是它的落点。

总体判断：旧基线的"ToolRegistry 物化/结算中心"架构已解体为**三权分立**——注册（Tool.Service + State/Editor）、执行（runtime.ts + Snapshot.execute）、结算（Step fiber + ToolOutput + publisher）。specs/v2/tools.md 描述的 `Tool.make`/`Tools.register` API 形态尚未在 Core 落地（Core 现为 editor.add / ctx.tool.transform），阅读规范时应以"语义不变量"对照而非字面 API。

## 相关篇

- [4_session.md](4_session.md)：Session/Step 与 durable 事件流（工具结算的宿主）
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md) §5：工具 fiber 并行执行时序（本篇不展开调度）
- [11_permission.md]（规划中）：权限评估细节、ask 规则、CorrectedError 语义
- `specs/v2/tools.md`（opencode 仓库内）：语义规范（部分条款为规范先行，见 §7/§8 漂移注记）

## 附录 A：术语表

| 术语 | 定义 | 出处 |
|---|---|---|
| Tool.Info | Core 唯一的本地工具结构值（name/input/output/execute/options） | schema/tool.ts:92 |
| effectiveName | namespace 展平 + 非法字符替换后的注册键/模型侧名 | runtime.ts:278 |
| CodeMode | 把多个工具折叠为单个 `execute` 工具的目录机制，保 provider tool 数量与 prompt 前缀缓存 | tool.ts:239 |
| direct 工具 | `codemode:false`、直挂 provider tool 列表的工具（现全部 13 个编辑/检索类内置都是） | tool.ts:234 |
| Snapshot | 请求级物化：definitions + codeModeCatalog + execute 单口 | tool.ts:51 |
| Tool.Error | 唯一合法的模型可见失败 tagged error | schema/tool.ts:61 |
| foreign typed failure | 叶子抛出的非 Tool.Error typed error，被 runtime 强转 | runtime.ts:32-36 |
| decline 隧道 | DeclinedError 以 defect 穿透叶子、在 executeTool 复现为 typed failure 的通道 | permission.ts:248 |
| producer 自限 | 工具自己截断并如实标注 `metadata.truncated`，通用有界化随之跳过 | tool-output.ts:68 |
| transform | `State.Transformable<Editor>` 的注册操作单元，同步可重放 | tool.ts:41 |
| ToolDefinition | `@opencode/ai` 的模型侧工具定义（name/description/inputSchema/outputSchema） | ai/schema |
| durable 先于副作用 | Tool.Called 出版完成后才开始本地执行；结果经 publisher 进 durable 事件流 | step.ts:103-104 |

## 附录 B：源码阅读路线

按依赖序（每站 30-60 分钟）：

1. `packages/schema/src/tool.ts`（102 行）——先立契约：Info/Context/Error/Content/Result。
2. `packages/core/src/tool/runtime.ts`（281 行）——definition/execute 与两条 JSON Schema 兼容补丁。
3. `packages/core/src/tool.ts`（332 行）——Service/Editor/Snapshot 三件套，对照 §3.3-3.5 读。
4. `plugin/read.ts` + `read-filesystem.ts`——最完整的"读"范式（授权、备选路径、指令注入）。
5. `plugin/write.ts` → `edit.ts` → `patch.ts`——edit 动作族与 diff 预览。
6. `plugin/shell.ts`——命令解析授权、jobs、后台流文件。
7. `tool/mcp.ts`（152 行）+ `mcp/index.ts` 的 `tools()/callTool` 段——桥接层。
8. `tool-output.ts`（150 行）+ `session/runner/step.ts:86-160`——结算与有界化闭环。
9. `specs/v2/tools.md`——回头对照规范与实现的差异点（§7/§8）。

验证方式建议：改任意一个 `editor.add` 的 description 跑 `bun run dev:live`，从 `/tools` 面板看注册结果；或在 §3.0 的链路上打断点观察 before hook 改写 input 的实际效果。全部行号基于 v2.0.30 dev（`e25b7abf89`），升级后请以 `wc -l` + `grep -n` 复测。
