# 上下文工程（Context Engineering）：把分散的上下文治理串成一张图

> 一句话：**opencode 没有使用"Context Engineering"这个词，但它的四个子系统——指令编排（6_context）、窗口治理（7_compaction）、工具输出有界化（9_tool §7）、输入准入协调（5_inbox）——恰好构成一份完整的上下文工程实现答卷；本篇用社区的问题框架把它们串成一张图，并审计一次 Step 的上下文成本、指出方案取舍与版图缺口。**
> 代码基线：v2.0.30，dev 分支（e25b7abf89）；引用行号实测（2026-10-26 复查，抽查清单见文末）。
> 可信度分级：**机制描述 = 实证**（逐处带文件:行号）；**§0 社区定义 = 转述**（2025 年社区流行语的语境归纳，非 opencode 官方说法）；**§3/§4 取舍与缺口评估 = 分析**（作者判断）。
> [返回 README](README.md) | 前置互链：[6_context.md](6_context.md)（指令编排）| [7_compaction.md](7_compaction.md)（窗口治理）| [9_tool.md](9_tool.md) §7（输出有界化）| [5_inbox-coordination.md](5_inbox-coordination.md)（准入协调）| [3_model-protocol.md](3_model-protocol.md) §5（缓存断点）

---

## ⚠️ 先辨析：本篇与 6_context.md 的关系

[6_context.md](6_context.md) 篇首已辨析过"上下文"一词的三义（UI 快照 / token 用量 / LLM 请求组装）。本篇再叠加一层辨析：

- **6_context.md 是"指令子系统深挖"**——它回答"Instructions Source 代数如何设计、三表如何持久化、epoch/delta 如何演进"，视角在**一个子系统内部**。
- **本篇是"统一视角收口"**——把指令编排、压缩、工具输出、输入准入四个子系统放到**同一个问题框架**（上下文工程）下对齐，看它们如何分工共治"模型该看到什么"。本篇不引入新机制、不重写任何单篇细节，所有机制一律互链。

一句话：想改指令系统去读 6_context，想理解四个系统为什么长这样、还缺什么，读本篇。

---

## 0. 什么是上下文工程（社区语境转述）

> ⚠️ 本节为社区语境转述，可信度：转述。opencode 代码库与 specs 中没有出现 "Context Engineering" 这个词——它是 2025 年 agent 工程社区流行起来的框架，一般认为由 Anthropic 的《Effective context engineering for AI agents》等文章带火，此前另有 "Context Engineering is the new Prompt Engineering" 的流行说法（据社区文章转述，未逐一考证原始出处）。

社区语境下的核心张力一句话可以说完：

```
context window 有限且贵  ×  agent 需要知道的东西越来越多
     (fixed, costly)              (instructions, env, history,
                                   tool results, memory, RAG...)
```

**窗口的完整构成**（Anthropic 官方文档实锤，platform.claude.com/docs "Context windows" 原文：*"Everything in the request counts toward the context window: the system prompt, every message in `messages` (including tool results, images, and documents), and your tool definitions. The output Claude generates for the turn, including its extended thinking, counts too."*）：

```
├─ system prompt（含 instructions——opencode 的 system[1] 基线）
├─ messages（对话 + 工具结果 + 图片/文档附件）
├─ tool definitions（工具 schema）
└─ ★本次生成的输出 + extended thinking（推理块也占窗口——
     这就是 anthropic thinking 要签名原样回放的原因之一）
```

即窗口 ≈ "system + instructions + messages + tools + **本次输出与 thinking**"——最后一样最常被漏算（18 篇 §2 成本审计已计入）。Anthropic 同页还给了 context rot 的官方定义：*"As token count grows, accuracy and recall degrade... This makes curating what's in context just as important as how much space is available"*——**治理（curation）比容量更重要**，这是整个上下文工程的存在理由。

由此衍生的问题清单（社区讨论的常见分法，转述归纳）：

1. **该放什么进窗口**（selection）——不是所有已知信息都值得占窗口；
2. **怎么省着放**（frugality / caching）——前缀缓存、分片、按需注入；
3. **环境变了怎么让模型知道**（freshness）——日期、文件、配置的变更感知；
4. **窗口满了怎么办**（compaction / summarization）——摘要替换前缀历史；
5. **工具结果防污染/防撑爆**（tool output bounding）——大输出截断、落盘、标记；
6. **用户输入何时进窗口**（turn-taking / steering）——插话与排队的语义；
7. **指令去重与稳定身份**（dedup / identity）——同一指令不重复占窗口，变更可检测。

两条社区反复强调的次级张力（转述归纳）：

- **窗口不是越大越好**：长窗口带来注意力稀释与 context rot（定义见上方官方引文）——所以"省着放"不只是省钱，也是质量手段；
- **上下文不只是 prompt**：Prompt Engineering 时代的对象是"这一次请求怎么措辞"；Context Engineering 的对象扩展到**整个 agent 生命周期里模型视野的构成**——指令、环境、历史、工具结果、外部知识的进/出/换/省全在内。这也是"Context Engineering is the new Prompt Engineering"一说的实际含义（转述）。

社区方案的常见形态：手写 prompt 模板 + 字符串拼接、显式的 context 管理框架（如 LangChain 的 memory 模块）、各家 agent harness 自带的 summarize 环节。opencode 的特殊之处在于：它把这七个问题**全部做成了 durable 的、有代数身份的机制**，而不是请求时的临时字符串操作——这正是下面总表要展示的。

社区语境下 agent 的上下文工程通常被描述成一个**循环**而非一次拼接（转述归纳）：

```
        ┌──────────────────────────────────────────────┐
        │                                              │
        ▼                                              │
  [感知/perceive] 收集候选：指令/环境/历史/工具结果/记忆   │
        │                                              │
  [选择/select]   决定哪些进窗口、以什么形态             │
        │                                              │
  [预算/budget]   体积/缓存/成本考量下的取舍             │
        │                                              │
  [执行/act]      组装请求 → LLM → 工具 → 新结果        │
        │                                              │
  [维护/maintain] 压缩/去重/失效更新/记忆沉淀 ───────────┘
```

对照 opencode：这个循环的每一段都有对应子系统（select/budget → 6_context；perceive 中的环境部分 → 内建指令源 watcher；maintain → 7_compaction + delta；act 期间的输入时序 → 5_inbox；act 期间的输出体积 → 9_tool §7）。缺的主要是"记忆沉淀"回边（§4）。

术语对照（社区词 ↔ 本笔记系列的词）：

| 社区语境 | 本系列用语 | 落点 |
|---|---|---|
| system prompt / instructions | 指令（Instructions Source、基线/增量） | [6_context.md](6_context.md) |
| compaction / summarization | 压缩（摘要替换前缀历史） | [7_compaction.md](7_compaction.md) |
| tool output truncation | 输出有界化（三层限界） | [9_tool.md](9_tool.md) §7 |
| steering / turn-taking | 投递词汇（steer/queue/控制项） | [5_inbox-coordination.md](5_inbox-coordination.md) §2 |
| prompt caching / cache breakpoints | 前缀缓存 / 缓存断点（4 断点预算） | [3_model-protocol.md](3_model-protocol.md) §5 |
| memory / RAG | （上游未覆盖，见 §4） | features/09-memory（fork） |

## 0.5 通俗完整描述（先读这段再看细节）

**本篇是收口篇**：opencode 里没有叫"上下文工程"的模块——它是把散在四个子系统里的上下文治理（指令编排/压缩/输出有界化/投递控制）用社区流行的问题框架串成一张图，回答"窗口这个又贵又容易腐烂的资源，这个系统是怎么管的"。

核心张力一句话：窗口有限且贵（还越大越烂——context rot），而 agent 需要知道的越来越多。由此八个子问题（放什么/怎么省/变了怎么通知/满了怎么办/工具结果防污染/输入何时进/怎么去重/缓存怎么打），每个问题 opencode 都有一个机制应答——本篇的映射总表就是这八行"问题→机制→落点"。

如果只带走一个视角：把窗口当成**预算管理的资源**而不是"越大越好的容器"——每次调用都在决定"这 token 值不值"，治理手段（基线缓存/增量公告/压缩/截断/时机控制）全是在提高每个 token 的信噪比。opencode 的特色是把这一切 durable 化（连指令变更都是事实）和代数化（Source 可组合）——比多数框架工程化，代价是理解门槛。

一句话：**八个问题×八个机制的一张映射表+一次调用的成本审计——"已有四篇的统一目录页"**。

---

## 1. 问题清单 → opencode 实现的映射总表

本篇灵魂。每一行：社区问题 → opencode 机制 → 落点篇节（机制细节一律互链，不重写）。

| # | 上下文工程问题 | opencode 机制 | 落点 |
|---|---|---|---|
| 1 | 该放什么进窗口 | **Instructions Source 代数**：七类源（CodeMode 目录/MCP/references/skills/discovery/builtins/entries）按"最共享在前"合并；每个源 key/read/render，值经 canonical JSON 哈希去重；`select/load` 两阶段解析出 system 两段 + 消息重放 | [6_context.md](6_context.md) §1/§3 |
| 2 | 怎么省着放（分片） | **epoch 基线 + chronological delta**：指令基线只发 system[1] 一份，中途变更以 delta 事件文本进消息流（"The instructions … changed. Here's the diff"），不重发全量基线 | [6_context.md](6_context.md) §1/§5 |
| 3 | 怎么省着放（前缀缓存） | 三层配合：① 基线排序稳定（"most to least shared"，context.ts:147-149 注释明示缓存目标）；② anthropic 协议 **4 断点预算**按失效序分配 tools→system→messages（anthropic-messages.ts:1044-1063 实测，超额丢 message-tail 并告警）；③ **CodeMode 折叠**把多数工具收进单个 `execute` 目录，压缩 tool schema 体积且保前缀稳定 | [6_context.md](6_context.md) §9；[3_model-protocol.md](3_model-protocol.md) §5；[9_tool.md](9_tool.md) §8 |
| 4 | 环境变了怎么让模型知道 | **内建指令源 + InstructionsUpdated 注入**：`core/date`（跨天产生 "Today's date is now: …" 增量）、`core/environment`（`<env>` 块变更增量）、AGENTS.md watcher（unified diff 增量）——全部走 observe→diff→durable 事件→消息流 | [6_context.md](6_context.md) §2/§3/§5 |
| 5 | 窗口满了怎么办 | **压缩三路触发**（manual 控制项 / auto 每步前 due() / overflow 拒绝恢复）+ **双态摘要 prompt**（首次/更新）+ **缩退循环**（SHRINK_STEPS 0.7→0.5→0.35，实测 compaction.ts:95）+ 原生 provider 窗口路径；完整 transcript 永远 durable，压缩只移动读取起点 | [7_compaction.md](7_compaction.md) §2-§4 |
| 6 | 工具结果防污染/防撑爆 | **三层限界**：producer 自限（read 2000 行/50KiB 分页、shell backing 文件，如实标 `metadata.truncated`）→ 通用有界化（`ToolOutput.truncate` 头保留 + `[showing lines 1-N of M; full output saved to <file>]` marker + 落盘 7 天）→ 压缩期二次截断（1250 字符） | [9_tool.md](9_tool.md) §7 |
| 7 | 用户输入何时进窗口 | **投递词汇表**：`steer`（安全步边界插话，批投递）/ `queue`（空闲边界恰好一个）+ compaction/move **控制项硬边界**（后续 steer 不跨）+ steered compaction 提队（保原文逐字进检查点之后） | [5_inbox-coordination.md](5_inbox-coordination.md) §2 |
| 8 | 指令去重与冻结 | **canonical JSON 哈希 + 三表 + delta 冻结**：sha256(canonical(value)) 内容寻址去重（instruction_blob 跨会话共享）；渲染文本冻结进 durable 事件（不事后重渲染）；Map 插入序即渲染序 | [6_context.md](6_context.md) §4/§9 |

一张图看四个子系统的分工地带：

```
                    一次 LLM 请求的窗口构成
 ┌───────────────────────────────────────────────────────────────┐
 │ system[0] harness 模板        │ 6_context §2（静态，缓存友好）  │
 │ system[1] 指令 epoch 基线     │ 6_context §5（有序，缓存前缀）  │
 │ messages:                     │                                │
 │  ├─ 用户/合成消息             │ 5_inbox §2（何时进窗口）        │
 │  ├─ InstructionsUpdated 增量  │ 6_context §5（变更感知）        │
 │  ├─ 历史（含压缩检查点展开）   │ 7_compaction §4（窗口治理）     │
 │  └─ 工具结果（有界化后）       │ 9_tool §7（防撑爆）             │
 │ tool schemas / CodeMode 目录  │ 9_tool §3.4 + 3_model §5（缓存）│
 └───────────────────────────────────────────────────────────────┘
 治理手段按位置分工：进窗口前（准入/发现/哈希）→ 进窗口时（组装/缓存）→ 窗口内（截断/marker）→ 窗口满（压缩/epoch 推进）
```

换个轴看——**治理的时间尺度**（同一机制表，按生效周期重排）：

```
 时间尺度     生效机制                                   归属
 ────────   ─────────────────────────────────────────  ───────────
 跨会话      blob 内容寻址（同一指令内容只存一份）          6_context §4
 会话生命周期 epoch 基线 + 压缩检查点（旧增量被基线吸收）  6_context §5 / 7_compaction §4
 多步之间     CodeMode 目录缓存、估算锚点（measured 双轨）  9_tool §3.4 / 7_compaction §3.6
 单步内       steer 批投递（一次 step 重置）、工具并发结算   5_inbox §2.5 / 9_tool §3
 单次请求     前缀缓存断点、截断 marker、delta 增量文本     3_model §5 / 9_tool §7 / 6_context §5
```

四个子系统一句话各表（详细机制见互链篇）：

- **6_context（指令编排）**：把"模型该知道什么"代数化、durable 化、可增量更新——回答"知识以什么形态存在"；
- **7_compaction（窗口治理）**：把无限增长的 transcript 折叠成有限窗口——回答"历史装不下怎么办"；
- **9_tool §7（输出有界化）**：把最凶的窗口膨胀源（工具输出）在入口处截住——回答"单条结果多大算够"；
- **5_inbox（准入协调）**：把"何时进入模型视野"变成显式词汇——回答"输入的时序主权归谁"。

## 2. 一次 Step 的上下文成本审计（走读视角）

从 model-request 组装到 step.ended 的 usage，跟着一个 Step 走一遍"窗口里都有什么、谁在省钱"。

```
Step 开始（runner/llm.ts:202-207 prepareContext，实测）
 │
 ├─ context.select ────────── 七类指令源 + 工具快照（权限收窄）
 ├─ InstructionState.prepare ─ observe → diff → InstructionsUpdated?（durable）
 ├─ compaction.compact(auto) ─ due()? 估算 ≥ ceiling 则先压缩再组装
 │                            （7_compaction §2.2 三道闸）
 ▼
model-request.ts baseTranscript 组装（:105-119，实测）
 │
 │  占窗口的部分                      省钱机制在旁边做什么
 │  ─────────────────────────────   ─────────────────────────────────
 │  system[0] harness 模板+工具指导   静态文本；随工具名集合变化，
 │                                   CodeMode 折叠使其稳定（9_tool §8）
 │  system[1] 指令基线                排序"最共享在前"（缓存前缀）；
 │                                   blob 内容寻址，跨会话只存一份
 │  messages（durable 重放）          压缩边界裁剪（history.ts，只发
 │                                   检查点之后的）；工具结果已过
 │                                   三层限界；InstructionsUpdated
 │                                   是增量文本不是全量重发
 │  tool schemas                     CodeMode 目录缓存跨 step 共享
 │                                   （tool.ts:246-254，名单不变则复用）
 ▼
provider 传输层（3_model-protocol §5）
 │  anthropic: 4 断点按失效序 tools→system→messages（实测 :1044-1063）；
 │  cache-policy.ts 按 provider 分叉（Gemini 只用最后断点等）
 ▼
step.ended → usage（step.ts:246-257，实测：finish 时随 publisher 发布 usage）
 │  input/output/cache token 与 cost 落 SessionTable 投影
 │  （4_session §6/§7 统计面）——压缩调用单独计费（UsageRecorded
 │   source="compaction"，7_compaction §3.4）
 ▼
下一个 Step 重复；估算锚点 = 最近一次 provider 实测（estimatePrompt
measured/estimated 双轨，7_compaction §3.6）
```

审计结论（分析）：

- **占窗口的四大件**里，唯一"无治理、全量进入"的是 harness 模板与指令基线本身——它们的治理方式不是截断而是**保持稳定**（换缓存命中）；6_context §10 已指出 AGENTS.md 全文无截断逻辑，超大指令的兜底是压缩的 overflow 路径。
- **省钱机制的时间尺度不同**：请求间靠前缀缓存（稳定排序+断点），会话内靠增量（delta）与截断（三层限界），会话生命周期靠压缩（epoch 吸收旧增量）。三层互不替代。
- **成本可见性闭环**：usage 从 provider 实测回流（step.ended），既是计费事实又是下一次压缩估算的锚点——"窗口贵"这个前提本身被 durable 地记录着。

再补一张"同一次请求里各部分的缓存命运"表（分析，基于上述实证机制推导）：

| 窗口成分 | 变更频率 | 缓存命运 | 谁在保它 |
|---|---|---|---|
| tool schemas（CodeMode 折叠后） | 极低（工具名单变才变） | 最稳定前缀；anthropic 断点最优先保它 | CodeMode + 4 断点按失效序分配 |
| system[0] harness 模板 | 低（版本级） | 随工具指导渲染变化 | 模板静态文本 |
| system[1] 指令基线 | 中（AGENTS.md 编辑/跨天跨 epoch） | epoch 内稳定；压缩时被新基线吸收重立 | "最共享在前"排序 + advanceEpoch |
| messages 前缀 | 每 Step 追加 | 尾部增长，前缀命中 | 消息尾断点（超额时最先被丢） |
| messages 尾部（含新 delta/工具结果） | 每次 | 每次必重算 | 无——这是增长的不可缓存部分 |

这张表解释了 6_context §9 诸多不变量的共同动机：**一切排序/冻结/折叠设计，都是在把"变更"往窗口尾部赶**——头部越稳定，缓存命中率越高。

一个三步演进算例（分析；数字为示意，机制均实证）：

```
Step 1（会话开局）
  system[0]  harness 模板（~1k tok）            → 缓存写
  system[1]  基线：date+env+AGENTS.md（~3k）     → 缓存写（断点护住）
  messages   [U1]（~0.1k）                      → 计费输入
  tools      CodeMode 目录（折叠后 1 个 execute + 少数 direct）
  ⇒ 输入小；缓存写一次，后续命中

Step 7（中途，AGENTS.md 被编辑过一次、跑过 3 个工具）
  system 两段不变                               → 缓存读（命中）
  messages   [U1..A6] + InstructionsUpdated diff 增量
             + 3 条有界化工具结果（最坏 50KiB/条 → 已截断）
  ⇒ 增长全在尾部：前缀命中，尾部按 input 价计费

Step 30（估算 ≥ ceiling，auto 压缩后）
  system[1]  新基线（advanceEpoch 吸收全部旧增量） → 缓存重写一次
  messages   [<checkpoint> 摘要+recent] + 其后消息  → 窗口回到 ~keep(15k) 量级
  ⇒ 代价：一次压缩调用计费（source="compaction"）+ 基线重立的缓存写；
    收益：窗口从 O(全史) 回到 O(摘要+近期)
```

算例点破的经济学：opencode 的省钱策略本质是**用少量的、可预测的"缓存重写事件"（epoch 推进、工具名单变化）换取常态的缓存读**——压缩的计费诚实（失败也计，7_compaction §9.7）正是让这笔交换可核算的前提。

## 3. opencode 方案的特点与取舍（分析）

**特点**（对照社区常见形态）：

1. **Source 代数化**：多数框架的"上下文管理"是请求时的字符串拼接；opencode 把"模型该知道什么"建成带 key/codec/render 的类型化代数 + 三张 SQLite 表 + 内容寻址哈希。收益是去重、变更检测、durable 重放全部有了机制基础；代价是 10 文件约 1,400 行的复杂度（6_context §0）。
2. **durable 化**：连指令基线/增量都是 durable 事件——"指令也是事实"，与消息同等待遇。社区方案里指令通常只是请求时的派生物。衍生保证：渲染文本冻结进事件（重渲染需要原始 Location 作用域的源，state.ts:45-47 注释）、fork 可播种基线、revert 可整体重置。
3. **epoch/delta 的缓存友好设计**：基线稳定排序 + 增量走消息流 + 最后一个 Step 保留工具定义（"preserve the provider's cached prefix"，llm.ts），把"模型知识更新"与"前缀缓存命中"这对天然矛盾工程化地调和了。
4. **治理词汇显式化**：steer/queue/控制项不是隐式行为而是 durable 表的列词汇（session-inbox.ts:11），投递语义可审计。

**诚实的简化处**（实证 + 分析）：

- **指令哈希只算 value，不算渲染文本**。实测 `hash(value)` = sha256(canonical JSON of value)（instructions/index.ts，`renderInitial`/`hash` 相邻定义）：若一次代码升级改了某源的渲染文案而 value 不变，diff 不产生 delta——消息流里模型看到的仍是旧冻结文本，直到 value 真变。这是"身份 = 值"的简化，渲染器版本不参与身份。
- **基线本身无体积治理**：AGENTS.md 全文进 system[1]，无截断（6_context §10）；api 上下文 KV 有 256KiB 硬限但发现链没有。
- **估算可以错，拒绝兜底**：本地 token 估算是启发式（4 字符/token），真正的裁决者是 provider 拒绝 + SHRINK_STEPS 缩退（7_compaction §9.5）——省钱机制里保留了一条"不相信自己估算"的务实退路。

特点与代价对照收拢：

| 设计选择 | 收益 | 代价 |
|---|---|---|
| Source 代数化 | 去重/变更检测/durable 重放有统一基础 | 10 文件约 1,400 行；理解门槛高（本系列需要一整篇才能讲清） |
| 指令 durable 化 | 冻结/回退/fork/重放全部免费获得 | 每次指令变更是 durable 事件（存储与 seq 消耗）；哈希只认 value 的盲区（上） |
| epoch/delta 分片 | 前缀缓存命中 + 增量可见性兼得 | epoch 语义渗透到三张表 + 投影 + 压缩结束钩子（advanceEpoch 的调用链跨三个文件） |
| 截断而非语义压缩（工具结果） | 确定性、零额外 LLM 成本、可落盘复查 | 有用信息可能被切掉；模型需主动 read 落盘文件找回 |
| 控制项硬边界 | 压缩/迁移时历史版本无歧义 | steer 可能延迟一轮投递（边界语义优先于即时性） |

## 4. 版图缺口（社区议题 × opencode 未覆盖/半覆盖）

> 可信度：分析。定性标注：✗ 未覆盖（上游）/ ◐ 半覆盖 / ✗✓ fork 自研。

| 社区议题 | opencode 现状 | 定性 |
|---|---|---|
| **长期记忆（memory）** | 上游 v2.0.30 无任何 memory 插件/机制（实测：`find packages -ipath "*memory*"` 仅命中环境内存基准等无关文件；instructions/entries 都是会话级）。用户 fork 的 `packages/opencode/src/plugins/memory/` 自研了四子系统（FTS5 检索/Checkpoint/Dream 整合/Distill 工作流发现，约 2,700 行）——见 [../features/09-memory/memory-feature.md](../features/09-memory/memory-feature.md)。注意 fork 基线与上游 v2.0.30 不同包结构，能力不互通 | ✗✓ |
| **RAG / 检索注入** | 无内建检索增强管道。最接近的机制是 instruction_entry 的 `api/<key>` KV 上下文（宿主可 put 任意 JSON，`<context key="…">` 渲染）与 glob/grep 工具（模型主动检索，非自动注入）——即"检索"是模型拉式而非系统推式 | ✗（拉式工具替代） |
| **工具结果的语义压缩** | 当前是**截断不是摘要**：实测 tool-output.ts 的 `truncate` 为头保留 + 落盘 marker（逐行累计字节，无 LLM 参与）；进入压缩摘要时也是 1250 字符硬截（`truncateToolOutput`）。摘要只发生在 compaction 的整窗层面，不针对单条工具结果 | ◐ |
| **多 agent 共享上下文** | subagent 是独立子会话（结论摘要交付），无共享窗口/黑板机制（社区议题，上游未涉及） | ✗ |
| **上下文预算分配** | 无"给哪类内容留多少 token"的显式预算；只有总量层面的 ceiling/keep/reserve（压缩参数）。社区讨论的 token budgeting per source 未实现 | ✗ |

缺口的共同形状（分析）：opencode 治理的都是**会话内**窗口（进什么、怎么省、满了压）；**跨会话**的知识沉淀（memory/RAG）整体留白——唯一跨会话的机制是 blob 内容寻址（省存储，不是给模型的知识）。这与"指令也是事实"的 durable 哲学并不冲突，只是尚未延伸到跨会话记忆那一层。

逐项补注（分析——"为什么算缺口"与"为什么可能是有意留白"各一句）：

- **memory**：算缺口，因为社区已把跨会话记忆视为 coding agent 标配；但也可能是有意留白——core 保持会话语义封闭（事件溯源聚合根），memory 涉及的全局读写会破坏"恢复只依赖数据库事实"的简洁性，fork 选择用插件层（hooks + 独立 SQLite）实现恰好避开了核心侵入。
- **RAG**：opencode 的立场近似"检索是模型的工具而不是系统的管道"——glob/grep/read 给模型主动检索能力，instruction_entry 的 KV 留了宿主推式注入的口子。是否算缺口取决于你站在"系统应自动供给知识"还是"模型应自己找"哪一边。
- **语义压缩**：截断+落盘 marker 的选择是确定性与成本的胜利；社区讨论的"LLLM 摘要工具结果"会引入不确定性与额外调用。半月覆盖的原因是压缩摘要路径（compaction prompt 的 Relevant Files ≤15 等节）已经在整窗层面做了语义取舍，单条结果层面再摘要的边际收益存疑。
- **多 agent 共享上下文**：subagent 摘要交付的模式与"独立上下文、显式通信"的社区最佳实践一致，共享窗口普遍被认为是反模式——这一项与其说缺口，不如说立场。
- **预算分配**：总量参数（auto/buffer/keep）存在但无 per-source 预算。这是 Source 代数预留了可能性（key 命名空间天然是预算单位）而未实现的部分。

## 5. 相关篇导航

```
                 ┌────────────────────────────┐
                 │ 19_context-engineering     │
                 │ 本篇：统一问题框架收口       │
                 └──────┬──────┬──────┬───────┘
        指令编排 │       │ 窗口治理 │      │ 输入准入
      ┌─────────┘       │         └──────────┐
      ▼                 ▼                    ▼
 6_context.md      7_compaction.md     5_inbox-coordination.md
 Source 代数/三表    三路触发/摘要/缩退    steer/queue/控制项
 epoch+delta/缓存   循环/边界三态        边界/投递状态机
      │                 │
      │  工具结果有界化   │ 缓存断点(§5.1 anthropic 4 断点)
      ▼                 ▼
  9_tool.md §7     3_model-protocol.md
  三层限界/marker   断点预算/provider 分叉
```

阅读顺序建议：本篇 §1 总表建立地图 → 按问题域进四篇深挖 → 回本篇 §3/§4 看取舍与缺口。若只关心"这一次调用带什么"，直接读 [6_context.md](6_context.md)。

四篇之间的互相引用关系（本篇之外，它们彼此也已互链）：

- 6_context ↔ 7_compaction：epoch 推进由 `Compaction.Ended` 投影驱动（6_context §5 / 7_compaction §4）——指令基线与压缩边界共享同一个事件；
- 6_context ↔ 9_tool：`Tool.Snapshot` 进 `context.select`、工具名进 system 模板指导（6_context §0/§2 / 9_tool §3.4）；
- 5_inbox ↔ 7_compaction：压缩作为控制项的投递边界语义（5_inbox §2.3 / 7_compaction §2.1）——同一现象的两个视角；
- 5_inbox ↔ 6_context：阻塞的初始基线让 inbox 挂起输入（6_context §10 / llm.ts 注释）——指令健康是输入投递的前置条件。

---

## 附：本篇抽查清单（2026-10-26，v2.0.30 dev e25b7abf89）

| # | 核实点 | 位置 | 结果 |
|---|---|---|---|
| 1 | baseTranscript system 两段组装 | session/model-request.ts:105-119 | ✅ 与 6_context §7 一致 |
| 2 | 工具输出是截断+落盘 marker，非摘要 | tool-output.ts:67-118 | ✅ 逐行字节累计，无 LLM 摘要 |
| 3 | anthropic 4 断点预算按失效序分配 | packages/ai/src/protocols/anthropic-messages.ts:1044-1063 | ✅ 注释原文 "Allocate the 4-breakpoint budget in invalidation order: tools → system → messages" |
| 4 | 指令哈希只算 value（text 不参与） | instructions/index.ts（hash 与 renderInitial 相邻段） | ✅ sha256(canonical(value)) |
| 5 | SHRINK_STEPS/RESERVE_MIN/UNKNOWN_WINDOW | session/compaction.ts:95-99 | ✅ [0.7,0.5,0.35] / 16_000 / 200_000 |
| 6 | step 结束 usage 随 publisher 发布 | session/runner/step.ts:246-257 | ✅ |
| 7 | 上游无 memory 插件 | `find packages -ipath "*memory*"` | ✅ 仅 environment/memory.ts 等无关文件 |
| 8 | core/date + core/environment 双内建源 | instructions/builtins.ts:23-47 | ✅ 渲染文案与 6_context §2 一致 |
| 9 | InstructionsUpdated durable 事件 | session/instruction-state.ts:49 | ✅ |
| 10 | Delivery 词汇 = steer\|queue | schema/src/session-inbox.ts:11 | ✅ |
| 11 | prepareContext 先于 Step、阻塞基线挂起输入 | session/runner/llm.ts:200-207 | ✅ 注释 "A blocked initial instruction baseline must leave admitted input pending" |
| 12 | fork memory 特性文档存在 | ../features/09-memory/（4 个 md） | ✅ 注明 fork 基线 |

## 相关篇

- [6_context.md](6_context.md)：指令编排深挖（本篇 §1 行 1-4/8 的展开）
- [7_compaction.md](7_compaction.md)：窗口治理深挖（本篇 §1 行 5 的展开）
- [9_tool.md](9_tool.md) §7：输出有界化（本篇 §1 行 6 的展开）
- [5_inbox-coordination.md](5_inbox-coordination.md)：准入协调（本篇 §1 行 7 的展开）
- [3_model-protocol.md](3_model-protocol.md) §5：缓存断点与 provider 分叉
- [../features/09-memory/memory-feature.md](../features/09-memory/memory-feature.md)：fork 自研记忆系统（本篇 §4 缺口呼应）
