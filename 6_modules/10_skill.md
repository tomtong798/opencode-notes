# 技能子系统详解（可发现的指令包：发现 → 注册表 → 双路径注入）

> 一句话：**Skill 是一个以 SKILL.md（或同名 .md）为载体的"指令包"——由 ConfigSkillPlugin 从配置目录/URL/内嵌插件三路发现并写入全局 Skill 注册表（State 编辑器 + `skill.updated` 事件），再经两条路径进入对话：用户 @/斜杠引用在 prepare 阶段展开为 `<skill_content>` 文本随 user 消息携带，模型自主选择走 skill 工具（opencode.tool.skill）；另有 SkillInstructions 把可用技能清单作为 instruction 注入 system 上下文。**
> 代码基线：v2.0.30，dev 分支（e25b7abf89）；行号实测（2026-10-19）。
> 权威出处：`packages/schema/src/skill.ts`、`packages/core/src/skill/`、`packages/core/src/config/plugin/skill.ts`
> [返回 README](README.md) | 前置 [6_context.md](6_context.md)（指令代数与三表） | 关联 [9_tool.md](9_tool.md) §4.8（skill 工具形态）、[../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)（prepare 阶段位置）

---

## 0.5 通俗完整描述（先读这段再看细节）

**Skill = 可发现的指令包**：一个目录放一份 SKILL.md（正文是你的操作指南，头部元数据声明名字/描述/触发词），模型看到技能清单后自主决定"这个任务该启用哪个技能"，启用后正文注入对话。你笔记库里那些 `.agents/skills/*/SKILL.md` 就是这个格式。

与相邻概念的分界：**agent 是"扮演谁"**（行为包，管全程）；**skill 是"怎么做某类事"**（操作手册，按需取用）；**插件是代码扩展**（跑 JS 改变行为）；**指令（AGENTS.md）是环境规范**（一直在场的背景）。技能=四兄弟里"按需加载的领域知识"那个。

三条注入路径（比直觉丰富）：①用户 @提及或斜杠——提交时展开成 `<skill_content>` 块拼进消息；②skill 工具——模型对话中自主调用（"我觉得该用 playwright 技能"）；③编程式激活——durable 事件入对话流。清单本身（有哪些技能可用）是指令系统的一个 Source——**只注入清单不注入正文**（省窗口），正文等真正启用才进。

发现来源三层：项目目录约定位置、配置显式指定（支持 URL/扩展路径）、插件注册——热重载（watcher+防抖）。远程技能下载有三重路径安全闸门。

一句话：**把"某类任务的操作知识"打包成可发现、可自主启用、按需注入的指令资产——清单常驻省窗口，正文按需再进场**。
---

## 0. 模块地图

```
packages/
├─ schema/src/
│   └─ skill.ts ................ 66  Info/ID/Name/Source 三态 + Updated 事件（skill.ts:27-65）
├─ core/src/
│   ├─ skill.ts ................ 133 ★门面：注册表（State 编辑器）+ available/prepare/toModelOutput
│   ├─ skill/
│   │   ├─ discovery.ts ........ 205  SkillDiscovery.pull：URL index.json → 原子下载/版本刷新
│   │   └─ instructions.ts ...... 96  SkillInstructions：技能清单 → instruction（delta 渲染）
│   ├─ config/plugin/
│   │   ├─ skill.ts ............ 193 ★ConfigSkillPlugin：source 计算 + 文件扫描 + watcher 热重载
│   │   └─ skill-file.ts ........ 63  SkillFile.parse：frontmatter → Skill.Info
│   ├─ plugin/skill.ts ......... 101  内置技能插件：opencode / report 两个 embedded 技能
│   ├─ tool/plugin/skill.ts ..... 72  skill 工具（→ 9_tool.md §4.8）
│   └─ session/
│       ├─ skill.ts ............. 16  SessionSkill.get：实例级解析（awaitActivation 后查注册表）
│       ├─ prompt.ts ........... 231  prepare L56-76：技能附件 → 展开文本
│       ├─ session.ts .......... 426  Handle.skill L226-246：发布 session.skill.activated
│       ├─ message-updater.ts .... —   activated 事件 → type:"skill" 消息（L174-186）
│       └─ runner/to-llm-message.ts — user.skills / skill 消息 → LLM user content（L271/L289）
└─ tui/src/prompt/codec.ts ..... —   TUI 把 @提及 编码为 SkillAttachment（一句带过）
```

`$ wc -l`：discovery 205 / instructions 96 / skill.ts 133 / tool/plugin/skill 72 / session/skill 16 / config/plugin/skill 193 / skill-file 63 / plugin/skill 101 / schema/skill 66。

抽查（sed 实测，佐证关键行号）：

```
$ sed -n '85,88p' packages/core/src/config/plugin/skill.ts
      for (const directory of directories) {
        add(Skill.DirectorySource.make({ type: "directory", path: AbsolutePath.make(path.join(directory, "skill")) }))
        add(Skill.DirectorySource.make({ type: "directory", path: AbsolutePath.make(path.join(directory, "skills")) }))
      }

$ sed -n '121,122p' packages/core/src/config/plugin/skill.ts
        .scan("{*.md,**/SKILL.md}", { cwd: directory, absolute: true, include: "file", symlink: true, dot: true })
        .pipe(Effect.orElseSucceed(() => [] as string[]))

$ sed -n '45,48p' packages/core/src/config/plugin/skill-file.ts
  const id =
    path.dirname(filepath) === directory && path.basename(filepath) !== "SKILL.md"
      ? path.basename(filepath, ".md")
      : path.basename(path.dirname(filepath))

$ sed -n '33,34p' packages/core/src/skill.ts
export const available = (skills: ReadonlyArray<Info>, permissions: Permission.Ruleset) =>
  skills.filter((skill) => Permission.evaluate("skill", skill.id, permissions).effect !== "deny")

$ sed -n '56,60p' packages/core/src/skill.ts
    path.basename(skill.path) === "SKILL.md"
      ? (yield* fs.scan("**/*", { cwd: directory, absolute: true, include: "file", dot: true }))
          .filter((file) => path.basename(file) !== "SKILL.md")
          .toSorted()
          .slice(0, 10)

$ sed -n '208,211p' packages/core/src/skill/instructions.ts
        const available = Skill.available(yield* skills.list(), permissions)
          .flatMap((skill) =>
            skill.description === undefined || skill.autoinvoke === false
              ? []

$ sed -n '270,271p' packages/core/src/session/runner/to-llm-message.ts
      const content = [
        ...(message.skills ?? []).flatMap((skill) => (skill.text === undefined ? [] : [Message.text(skill.text)])),

$ sed -n '289,290p' packages/core/src/session/runner/to-llm-message.ts
    case "skill":
      return [Message.make({ id: message.id, role: "user", content: message.text, metadata: message.metadata })]

$ sed -n '229,234p' packages/core/src/session/session.ts
    const session = yield* get(sessionID)
    const skill = yield* SessionSkill.get({ session, skill: input.skill }).pipe(
      Effect.provideService(Instance.Service, instances),
    )
    yield* bus.publish(
      SessionEvent.Skill.Activated,

$ sed -n '9,16p' packages/core/src/session/skill.ts
export const get = Effect.fn("SessionSkill.get")(function* (input: { session: Session.Info; skill: Skill.ID }) {
  const instances = yield* Instance.Service
  const skills = yield* Plugin.awaitActivation.pipe(Effect.andThen(Skill.Service), instances.provide(input.session))
  const skill = yield* skills.get(input.skill)
  if (!skill) return yield* new SkillNotFoundError({ skill: input.skill })
  return skill
})
```

## 1. 心智模型

Skill 不是"可执行插件"，而是**纯文本指令包**：一个 markdown 正文 + 少量 frontmatter 元数据，可能附带同目录资源文件（scripts/、reference/）。运行时的全部行为 = 发现（写注册表）→ 匹配（谁触发）→ 注入（变成模型可见文本）。

```
  配置目录 (.opencode/skill|skills/)          URL (index.json)         插件 embedded
        │ scan "{*.md,**/SKILL.md}"              │ SkillDiscovery.pull       │ ctx.skill.transform
        └───────────────┬────────────────────────┴───────────────────────────┘
                        ▼
          ConfigSkillPlugin.refresh（去重、watcher 热重载）
                        ▼
          Skill 注册表（State 编辑器 Map<ID,Info>，skill.updated 事件）
                        │
        ┌───────────────┼────────────────────────────┐
        ▼               ▼                            ▼
  SkillInstructions   SessionPrompt.prepare       skill 工具
  （清单进 system     （用户 @/斜杠引用 →          （模型按描述自选 →
   instruction，       Skill.prepare 展开为          权限断言 + 同一
   delta 更新）        <skill_content> 随           prepare 展开文本）
                       user 消息携带）
                        │                            │
                        ▼                            ▼
              to-llm-message: user.skills → Message.text    tool result content
              （或 session.skill.activated → type:"skill" 消息 → user content）
```

两条注入路径产出的文本**形状相同**（instructions.ts L162 明说："the same shape the skill tool returns"），因此模型侧无需区分来源。

## 2. 技能发现

### 2.1 三类 Source（schema/skill.ts:14-45）

```
Source = DirectorySource { type:"directory", path }   # 本地目录
       | UrlSource      { type:"url", url }           # 远端 index.json
       | EmbeddedSource { type:"embedded", skill }    # 插件直接注册的 Info
```

### 2.2 ConfigSkillPlugin（config/plugin/skill.ts）——目录/URL 的统一发现器

source 计算逻辑（L77-103）：

- **隐式目录**：对每个配置目录 entry，追加 `<dir>/skill` 与 `<dir>/skills` 两个 DirectorySource（L85-88）——即 `.opencode/skill/` 与 `.opencode/skills/` 都有效，去重靠 `Skill.Source.equals`（L80）。
- **显式配置**：`skills: string[]`（schema/config.ts:88-89，"Additional paths or URLs to discover skills from"）。http(s) URL → UrlSource；`~/` 展开（L94）；相对路径按 location.directory 解析（L98）。

扫描（load，L105-147）：对每个目录 `fs.scan("{*.md,**/SKILL.md}")`（L121）——顶层任意 .md 与任意深度的 SKILL.md；symlink 跟随，扫描失败静默为空数组。每个文件经 `SkillFile.parse`，Skipped 仅 logDebug（L129-136）。

### 2.3 URL 发现：SkillDiscovery.pull（skill/discovery.ts）

- 拉 `<url>/index.json`，Schema 校验为 `{skills: [{name, version?, files}]}`（L48-56）；失败 logError 返回空（L100-103）。
- **安全闸门**（全部不过即整技能丢弃）：name 必须是安全路径段（L16-25）；files 必须含 `SKILL.md` 或 `<name>.md`（L112）；每个 file 必须是安全相对路径、解析后同源、目标仍在技能根内（L27-46, L124-134）——防目录穿越/协议逃逸。
- 缓存位置：`<global.cache>/skills/<Hash.fast(url)>/<name>/`（L106）。版本刷新时序：

```
version 变化时（L157-192）：
  下载全部 files → staging = root.tmp-<uuid>/
      │ 任一失败 → 放弃（ensuring 清理 staging，旧版本继续可用）
      ▼
  校验 staging 有 SKILL.md 或 <name>.md → 写 .opencode-version
      ▼
  uninterruptible:
      root → backup(.old-<uuid>)      # 先挪走旧版
      staging → root                   # 原子换入
      失败 → backup → root 回滚后 re-fail
      成功 → 删除 backup
```
- **版本刷新协议**（L147-192）：无 version 或 version 未变 → 直接补下载；version 变 → 下载到 `staging`（`.tmp-<uuid>`），校验技能主文件存在后写 `.opencode-version`，再在 uninterruptible 段做 `root→backup → staging→root` 双 rename，失败回滚 `backup→root`（L173-187）。这是典型的原子目录交换。

### 2.4 热重载

- 每个目录/文件注册 FS watcher（watchDirectory L53-75，含"目录尚不存在时先 watch 最近存在祖先"的 firstMissing 逻辑 L47-51）；事件汇入 sliding PubSub，**100ms debounce** 后 `refresh(file) → ctx.skill.reload()`（L169-176，注释解释：编辑器保存会爆发事件，先订阅再防抖防丢）。
- `config.updated` 事件同样触发 refresh + reload（L181-191）。refresh 全量重建 Map<ID, Info>，**同 ID 后写覆盖先写**（L152-155）——即隐式目录与显式配置的冲突靠 source 迭代顺序 + Map 覆盖解决。

## 3. 技能结构

### 3.1 Info（schema/skill.ts:26-34）

```ts
Info = { id: ID, name: Name, description?: string,
         autoinvoke?: boolean, path: AbsolutePath, content: string }
```

无参数 schema——技能不是函数，没有 argumentSchema；"调用"一次即全量注入正文。

### 3.2 SKILL.md 解析（config/plugin/skill-file.ts:39-63）

frontmatter 仅四个字段：`name` / `description` / `metadata` / `disable-model-invocation`（L9-14）。

**ID 推导规则**（L45-48）：顶层 `<name>.md`（非 SKILL.md）→ 文件名去 .md；否则（任意深度的 SKILL.md）→ 父目录名。name 缺省 = id。

```
.opencode/skills/
├─ leetcode-reviewer.md          → id "leetcode-reviewer"（顶层非 SKILL.md）
├─ xlsx/
│   └─ SKILL.md                  → id "xlsx"（父目录名）
└─ shared/
    └─ programming/
        └─ SKILL.md              → id "programming"（**/SKILL.md 任意深度）
```

**autoinvoke 三态**（L49-51）：`metadata["opencode/autoinvoke"]` 优先；否则 `disable-model-invocation` 取反；宽容解析 true/yes/on/1（L29-37）。autoinvoke=false 的技能不进模型自选清单（见 §5），但仍可被用户显式引用/工具加载。

### 3.3 展开格式：toModelOutput（skill.ts:36-53）

```
<skill_content name="...">
# Skill: <name>
<正文>
Base directory for this skill: <dir>
Relative paths ... relative to this base directory.
<skill_files> <file>…（最多 10 个，L62 slice(0,10)，"file list is sampled"）</skill_files>
</skill_content>
```

`prepare`（L55-68）：仅当文件名是 SKILL.md 才扫描同目录资源文件（独立 .md 技能无附带文件），排除自身，排序取前 10。技能附带脚本/参考文件靠 "Base directory" 提示模型自行读。

### 3.4 权限面

技能作为 permission 资源：`Permission.evaluate("skill", skill.id, ruleset)` 过滤可用集（skill.ts:33-34）；skill 工具执行时 `permission.assert({action:"skill", resources:[skill.id]})`（tool/plugin/skill.ts:51-58）。另外 ConfigPlugin 的 ManagedPolicy 可按 `integration.use: skill:<id>` 直接从注册表移除技能（config/plugin/policy.ts:42-45）。

## 4. 触发与注入

### 4.1 路径①：用户显式引用（prepare 阶段）

TUI 侧把 @/斜杠提及编码为 `PromptInput.SkillAttachment {id, mention?}`（schema/prompt-input.ts:23-27；tui/src/prompt/codec.ts:24-26 一句带过）。

SessionPrompt.prepare（session/prompt.ts）：

- L40-51：`session.prompt` hook 先跑，**插件可增删改 skills 列表**（structuredClone 后的 prompt 一部分）。
- L56-76：对每个附件查 `Skill.Service.get(id)`；查不到 → `SkillNotFoundError`（L66，prepare 整体失败，进 inbox 前就被拒）；命中则 `Skill.prepare(fs, skill)` 展开为 text，去重缓存 prepared Map（L60/63-64，同 id 多次提及只展开一次）。
- L84：结果挂在 `Prompt.skills: {id, name, text, mention?}[]` 随 UserPayload 准入。

注入后模型实际收到的 user content 形状（to-llm-message L270-277 拼装）：

```
content = [
  Message.text("<skill_content name=\"xlsx\"># Skill: xlsx\n…\n</skill_content>"),  ← 每个引用技能一块，排在正文前
  Message.text("帮我处理这个表格"),                                                  ← 用户正文
  …附件块…
]
```

投影与消费：projector 把 payload.skills 原样写入 user 消息（projector.ts:636）；`to-llm-message.ts` L270-271 把每个 `skill.text` 作为**独立 Message.text 块排在用户正文之前**，拼进同一条 user role content——不是 synthetic 消息，是 user 消息的内部 content 块。压缩时同样保留为 `[Skill activated: <name>]\n<text>`（compaction.ts:767-771）。

### 4.2 路径②：模型自选（skill 工具）

工具形态归 [9_tool.md](9_tool.md) §4.8：输入仅 `{id}`；执行 = `skills.get` → `permission.assert` → 同一个 `Skill.prepare`；输出 content 即 `<skill_content>` 全文。工具描述明说"id 必须在可用清单中或被用户显式引用"——可见性由 §5 的 instruction 清单约束。

### 4.3 路径③（编程式）：session.skill.activated

`Session.Handle.skill`（session/session.ts:226-246）：`SessionSkill.get`（session/skill.ts:9-16——在 `Plugin.awaitActivation` 之后从**实例级** Skill.Service 取，确保该 session 的 location 插件已注册完）→ 发布 `session.skill.activated` durable 事件（携带 id/name/text）→ 可选 `execution.resume` 唤醒执行。事件经 message-updater L174-186 投影为 `type:"skill"` 消息（schema/session-message.ts:103-109），消费时按 user role、content=text 注入（to-llm-message.ts:289-290）。这是 API/SDK 驱动的"激活即消息"路径（session.ts:432 门面暴露）。

## 5. 技能与指令缓存

SkillInstructions（skill/instructions.ts）**不注入技能正文**，只注入"可用技能目录"：

- `load(permissions)`（L206-224）：`Skill.available` 过滤 deny → 剔除无 description 与 autoinvoke===false 的技能（L208-212——没描述就无法匹配，禁自选就不给模型看）→ 按 id 排序 → `Instructions.make` 注册到 key `core/skill-guidance`，codec 为 JSON 数组。
- 渲染：initial 是 `<available_skills><skill><id/><name/><description/></skill>…` 清单 + 使用说明（L158-166），其中 L162 明确告知模型"用户已显式注入的技能不必再调工具"。
- **delta 机制**：update（L168-191）用 `Instructions.diffByKey` 按 id 比对——仅增删时输出小 delta（"New skills…" / "IDs no longer available must not be used"），name/description 变化或增删并存则**重发全量清单并声明取代旧列表**（L176-180）。
- 挂载点：SessionContext.select（session/context.ts:137 `skills: skillInstructions.load(permissions)`），与 mcp/references/discovery 等 instructions 一起 `Instructions.combine`（L148-157），排序原则是"最跨会话共享的靠前以保 prompt-cache 前缀"（context.ts 注释）。permission 来源 = agent 权限 merge session 权限（L127），注释明说"Session permissions narrow discovery the same way they narrow the tool snapshot"。
- 三表持久化机制本体（entry/blob/state、epoch、observe/commit/apply）归 [6_context.md](6_context.md)，此处仅是其中一个 instruction source。

清单 initial 渲染样例（instructions.ts L149-166）：

```
Skills provide specialized instructions and workflows for specific tasks.
Use the skill tool to load a skill when a task matches its description.
The user may also invoke a skill directly. …
<available_skills>
  <skill>
    <id>xlsx</id>
    <name>xlsx</name>
    <description>Use this skill any time a spreadsheet file is …</description>
  </skill>
  …
</available_skills>
```

## 6. 会话技能状态

技能本身**无会话内激活状态机**——没有"activated set"，注入即生效。状态体现在三个投影点：

1. **注册表事件**：`skill.updated`（ephemeral，schema/skill.ts:36-37）在每次 State transform/reload 后广播（skill.ts:113），TUI 靠它刷新技能对话框/补全。
2. **会话事件**：`session.skill.activated`（durable，session-event.ts:288-298）→ `type:"skill"` 消息进读模型，随消息持久存在；revert/transfer 均按普通消息处理（transfer.ts:253 对其 text 做脱敏）。
3. **user.skills 附带**：显式引用的技能文本随 user 消息持久（projector.ts:636；transfer.ts:233-238 逐字段脱敏）。

生命周期一句话：注册表热更新随时可变，但**已进入消息流的技能文本是冻结快照**——清单 instruction 通过 delta 通告"某 ID 不再可用"，而非改写历史。

## 7. 插件注册面

插件通过 `ctx.skill` 域操作注册表（plugin/effect/skill.ts SkillDomain：`transform(SkillEditor)` + `reload()`；SkillEditor = list/get/add/update/remove，L5-15）。宿主侧桥接在 plugin/host.ts:464-468。要点：

- **没有 `skill.transform(input)` 之类的逐技能改写钩子**——插件只有编辑器式注册面（对比 tool 的 `ctx.tool.transform`）。
- 内置示例 `opencode.skill` 插件（plugin/skill.ts:95-120）用 `ctx.skill.transform` 注册两个 embedded 技能：`opencode`（opencode.md，路径伪 `/builtin/opencode.md`）与 `report`（report.md + 运行时诊断快照 L122-143）。
- 实例级注册：`SessionSkill.get` 在 `Plugin.awaitActivation` 之后经 `instances.provide(session)` 取该 location 的 Skill.Service（session/skill.ts:12）——**技能注册表是 per-location 实例的**，不同 workspace 可见不同技能集。
- plugin 宿主机制（加载/激活/host 桥接全貌）归 13_plugin.md（待写/在写）。

## 8. 不变量与设计取舍

- **技能即文本**：无参数、无代码执行面；一切"逻辑"都是让模型去读 Base directory 下的文件。
- **同一展开格式**：用户引用、工具调用、activated 消息三路共用 `toModelOutput`（skill.ts:36 + tool/plugin/skill.ts:27 + instructions.ts L162 的说明），模型侧统一。
- **注册表后写覆盖**：同 ID 冲突不报错，refresh 的 source 迭代序 + Map.set 决定胜者（config/plugin/skill.ts:152-155）——换取热重载的简单性。
- **文件列表采样上限 10**（skill.ts:62）：控制注入体积，注释 "file list is sampled" 直接写进模型可见文本（L46），让模型知道列表不完整、需自己 ls。
- **URL 技能的防御性**：discovery.ts 的段/路径/同源三重校验 + staging/backup 原子换目录——远端内容视为不可信。
- **缓存前缀优先**：技能清单 instruction 放在 combine 序列前部（context.ts:148-157 注释），因为它跨会话最稳定。
- **autoinvoke 默认开**：只要写了 description 就可被模型自选；opt-out 靠两代 frontmatter 字段兼容。

## 9. 失败与恢复

| 故障 | 行为 | 出处 |
|---|---|---|
| frontmatter 不合法 | Skipped(frontmatter)，logDebug 带具体 SchemaIssue，其余文件继续 | skill-file.ts:42-43, L129-135 |
| 非 markdown / 无 frontmatter | Skipped(markdown)，静默跳过 | skill-file.ts:40-41 |
| index.json 拉取失败 | logError，返回 []，该 URL source 为空集 | discovery.ts:100-103 |
| 单文件下载失败 | download 返回 false；version 刷新路径下任一失败即放弃本次（staging 被 ensuring 清理），旧版本保留 | discovery.ts:78-89, L167+190 |
| 换目录 rename 失败 | backup→root 回滚后 re-fail | discovery.ts:177-184 |
| 引用不存在的技能 id | SkillNotFoundError，prepare 失败，prompt 不准入 | prompt.ts:66；session/skill.ts:13 |
| 工具加载被 deny | permission.assert 抛错 → ToolFailure "Unable to load skill" | tool/plugin/skill.ts:29-30, 51-60 |
| 目录被删/新增 | watcher（含祖先 watch）→ debounce → refresh + reload | config/plugin/skill.ts:169-176 |
| 技能从注册表消失 | 清单 instruction 发 removed："Do not use any previously listed skill" | instructions.ts:221 |

名称冲突（同 ID）不是错误而是覆盖（§8），因此"冲突恢复"= 下次 refresh 的最后一次写胜出。

## 演进注记

旧笔记 4.6（2026-08-03，V1）：skill = `core/src/skill.ts` + `opencode/src/skill`，技能存于 **InstanceState**。现码（v2.0.30）差异：

- V1 的发现逻辑已迁移为插件化：`config/plugin/skill.ts`（ConfigSkillPlugin）承载全部目录/URL 发现 + watcher；`skill/discovery.ts` 只剩纯下载器。
- 技能注册表从 InstanceState 演进为独立 Effect 服务 `@opencode/Skill`（skill.ts:87），但保留了 per-location 实例性（`makeLocationNode`，skill.ts:129；session/skill.ts:12 经 instances.provide 取）。
- 新增 v2 机制：skill 工具（模型自选）、SkillInstructions 清单 instruction（delta 渲染）、`session.skill.activated` 事件消息、URL 技能的版本化原子刷新。
- `opencode/src/skill` 目录已不存在（v2 为 packages 结构）。

## 相关篇

- [6_context.md](6_context.md)——指令代数与三表（本文 §5 的底座）
- [9_tool.md](9_tool.md) §4.8——skill 工具的输入/权限/输出形态
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)——prepare 在执行链中的位置（本文 §4.1 展开 L56-76 细节）
- [2_config.md](2_config.md)——`skills` 配置项与目录布局
- [13_plugin.md](13_plugin.md)——插件宿主与 ctx 域（本文只写 ctx.skill 注册面）
