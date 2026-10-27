# 模型与协议层详解（models-dev 目录 → Provider/Model 解析 → llm.stream → 四轴 Route 协议适配的完整 LLM 出站管线）

> 一句话：**模型与协议层是一条"目录数据 → 运行时解析 → 统一请求 → 协议适配"的出站管线——core 侧用 models.dev 快照构建 Provider/Model 目录并解析出 `LanguageModel`，ai 包用 Protocol（语义契约）+ Endpoint/Auth/Framing/Transport（部署四轴）把统一的 `LLMRequest` 降级成 provider 原生请求，再把各家 SSE 流归一化为统一的 `LLMEvent` 流回吐。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19）。
> 权威出处：`specs/v2/provider-policy.md`
> [返回 README](README.md) | 前置 [1_data-layer.md](1_data-layer.md) | 关联 [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md) §5（Step 调用侧时序）

---

## 0. 模块地图

```
packages/core/src/                     ← 目录与解析（"选哪个模型"）
├─ models-dev/
│   └─ snapshot.txt      ★ 5.7 MB bundled 快照（models.opencode.ai/api.json，1 行）
├─ models-dev.ts         ★ 目录服务：快照解码/normalize/KV 缓存/定时刷新（450 行）
├─ model.ts              ★ Model 聚合状态：get/all/available/default/small（324 行）
├─ model-resolver.ts     ★ 目录 Info → 运行时 LanguageModel（484 行）
├─ variant.ts              thinking 档位展开（effort/toggle/budget_tokens，677 行）
├─ provider.ts             Provider 目录/包加载/Snapshot（532 行）
└─ aisdk-native.ts         @ai-sdk/* 老包桥接（329 行）

packages/ai/src/                       ← 协议与执行（"怎么发请求"）
├─ llm.ts                ★ 门面：generate/stream/generateObject（196 行）
├─ provider.ts             Provider 定义 helper（36 行）
├─ provider-error.ts     ★ 错误分类与可重试判定（339 行）
├─ cache-policy.ts         缓存断点预算（199 行）
├─ effort-updates.ts       effort 标记注入（~200 行）
├─ tool-history.ts         工具历史消息归一（98 行）
├─ tool.ts                 内部 Tool 定义（272 行）
├─ schema/                 ★ 规范数据模型（5 文件 1,772 行：events 742/messages 450/options 374/errors 177）
├─ route/                ★ 执行骨架（11 文件）
│   ├─ client.ts         ★ Route/LLMClient：compile→transport→状态机（696 行）
│   ├─ protocol.ts       ★ Protocol 接口：body.from + stream 状态机（88 行）
│   ├─ endpoint.ts / auth.ts / framing.ts   部署三轴
│   └─ transport/          http.ts / websocket.ts / websocket-channel.ts
├─ protocols/            ★ 协议实现（46 文件 15,032 行 + utils/ 24 文件 2,359 行）
│   ├─ anthropic-messages.ts ★ 1,676 行   ├─ open-responses.ts ★ 1,509 行
│   ├─ openai-chat.ts       ★ 1,322 行   ├─ gemini.ts        ★ 976 行
│   ├─ bedrock-converse.ts    895 行     ├─ openai-responses.ts 333 行
│   ├─ (alibaba/meta/xai/zai/mistral/cohere/venice… 各家 chat/messages/responses 皮肤)
│   ├─ (media 协议：images/speech/transcription/video 各家)
│   ├─ shared.ts              降级共用工具（418 行）
│   └─ utils/
│       ├─ tool-schema.ts   ★ 工具 schema 投影（112 行）
│       ├─ gemini-json-schema.ts  Gemini schema 净化（77 行）
│       ├─ tool-stream.ts     工具参数流式累积（251 行）
│       └─ (partial-json / responses-checkpoint / cache / lifecycle …)
├─ providers/              66 个部署文件：Route.make 组合（deepseek/openrouter/azure…）
└─ (image/speech/media/video/transcription 各模态客户端)

packages/llm/                          ← 空壳（仅 node_modules，无源码；本篇不涉）
```

core 侧另有接缝文件：`session/model-request.ts`（439 行，组装 `LLMRequest`）、`session/runner/step.ts`（消费 `LLMEvent` 流）、`session/runner/retry.ts`（115 行，重试策略）。

providers/ 目录的 66 个文件是"部署皮肤"的典型样例——每个平均不足百行，只声明五轴并按需覆盖，例如 `providers/deepseek.ts` 全文要点：`protocol: OpenAIChat.protocol` + `Endpoint.path("/chat/completions")` + `configure()` 里 `AuthOptions.bearer(input, "DEEPSEEK_API_KEY")`；`anthropic-compatible.ts`、`openrouter.ts` 等则各自组合多路由。协议本体（protocols/）与部署（providers/）的分界是本层最重要的代码组织决策。

## 0.5 通俗完整描述（先读这段再看细节）

<!-- https://models.dev/ -->
<!-- https://models.dev/api.json -->

用户选一个模型（如 `anthropic/claude-sonnet-4-5`）之后，这条"出站管线"开始工作。它要回答的问题只有一个：**怎么把一次统一的 LLM 请求送到 ~50 家长相各异的供应商那里，再把流式响应原样吐回来**。

**第一步：知道有哪些模型（目录）**。opencode 不在代码里硬编码模型清单——启动时从外部源 `models.opencode.ai/api.json` 拉取目录数据（离线兜底一份内置 snapshot，再往 KV 缓存里存），归一化成 `Snapshot`（每家 provider 的信息 + 模型列表 + limits）。core 侧的 Provider/Model 服务把目录按"有凭据、没被策略禁"过滤成可用清单——这就是 UI 模型选择器的数据来源。

**第二步：把目录条目变成可执行对象（解析）**。`ModelResolver.resolve(ref)` 做"查目录 → 叠加 variant（thinking 档位等 overlay）→ 取凭据 → **按目录数据指定的包加载 provider 文件**"。关键设计：协议选择既不硬编码也不靠用户配——目录数据里每个模型的 `package` 字段指向 `@opencode/ai/providers/<x>`，那个文件的 `Route.make` 组合决定一切。产出 `LanguageModel{provider, route, defaults}`——**从这一步起，"模型"不再是一条数据，而是一个绑定了完整路线（协议+端点+鉴权+传输）的可执行对象**。

**第三步：组装统一请求**。Session 侧（Step 执行时）把 system prompt + 对话历史 + 工具定义 + agent 设置组装成统一的 `LLMRequest`——**这是内部标准格式，与任何供应商无关**。工具 schema 统一转成标准 JSON Schema，缓存策略/推理力度等也在这一步应用。

**第四步：协议降级（ai 包的核心工作）**。`route.body.from(LLMRequest)` 把统一请求翻译成供应商原生格式——anthropic 的 messages、openai 的 chat、gemini 的 contents 各不相同，工具调用的表达方式也不同（四套协议适配器就是干这个的）。同一套 SSE 字节流回来，再被各自的事件归一化器统一成内部 `LLMEvent` 流（text-delta/tool-call/finish…）——**进来的路和出去的路对称：统一→方言，方言→统一**。

两个容易理解偏差的关键点：

1. **"模型目录"是数据不是代码**——加一家新供应商默认只是目录数据的事（改 models.dev），只有新协议形态才需要写代码。目录带版本兜底（snapshot + KV 缓存 + 5 分钟去抖刷新），断网也能用旧目录。
2. **协议 ≠ 供应商**——协议是"API 长什么样"（四种：anthropic-messages/openai-chat/open-responses/gemini），供应商是"部署在哪"（endpoint/auth/transport 部署四轴）。~50 家供应商靠"协议复用 + 各自 Route 组合"覆盖，而不是 50 套适配器。deepseek 走 openai-chat 协议、azure 是 anthropic 协议换端点换鉴权——同一协议代码服务多家。

理解这个分层的一个好框架：**这层本质是双向 Adapter 模式**（统一 ↔ 方言的转换），但有两个容易低估的地方：

- **适配的不只是"格式"，是"格式 + 语义"**。纯字段改名只是一半工作量，难的是语义对齐：工具调用四家四种表达（anthropic 的 tool_use 块 / openai 的 tool_calls / responses 的 function_call item / gemini 的 functionCall）、思维链字段形态各异（含 thought signature 回传）、流式分帧差异（openai 系 `[DONE]` 哨兵 vs anthropic 事件类型 vs 不完整帧拼接）——每个协议内部各有一个**事件状态机**，把各家 SSE 的语义结构（而非仅字节）归一成统一 LLMEvent 流。
- **Adapter（Protocol）之上还有一层"配置工厂"（Route）**。如果只有四种 adapter，~50 家供应商每家还得手写"选哪种 adapter + 什么端点 + 怎么鉴权"——Route.make 把这个组合本身数据化（Protocol + Endpoint + Auth + Transport 四轴），目录的 `package` 字段指向各家的 Route 组合。"选哪个 adapter"不再散在代码里，才有"加供应商默认只改数据"的效果。

失败与重试也有统一语义：provider-error.ts 把各家错误分类（限流/传输/内容策略/不完整流），`isRetryable` 供 Step 的重试协调器判定（限流退避重试、内容策略不重试）——Session 侧不需要理解各家错误码。

一句话走完全程：**目录说"有什么"，解析把条目变成"绑好路线的对象"，Session 组装"统一请求"，协议层翻译成"方言"发出去、把"方言响应"翻译回流——上层只见统一的模型和统一的事件流**。

---

## 1. 心智模型

整条管线的分界线在 **catalog（目录）与 runtime（运行时）之间**：core 的 `Model.Info` 只是目录里的静态描述（含 variants/cost/limit），真正发请求前必须经 `ModelResolver` 变成 ai 包的 `LanguageModel`——后者绑定了完整的 `Route`（protocol+endpoint+auth+transport）。协议选择**既不按 provider 硬编码也不按用户配置**，而是目录数据（`model.package` 指向 `@opencode/ai/providers/<x>`）决定加载哪个 provider 文件，该文件的 `Route.make` 组合决定了协议。

```
外部源                     core 目录域                    core 解析域                ai 执行域
────────                  ──────────────                 ────────────               ────────────
models.opencode.ai ──┐
  (api.json)         ├─→ ModelsDev.Service ──→ Provider.Service ──→ Model.Service
bundled snapshot.txt ┘    normalize():           (available 过滤/       (byProvider 索引/
                          SourceProvider→          凭据门控)             default/small)
                          Snapshot{info,models}
                                                       │ resolve(ref)
                                                       ▼
                                                  ModelResolver
                                                  withVariant()  ← thinking 档位 overlay
                                                  fromCatalogModel()
                                                       │ loadPackage("@opencode/ai/providers/deepseek")
                                                       ▼
                                              LanguageModel{provider,route,defaults}
                                                       │
     SessionModelRequest.prepare ──────────────────────┤
     (system+messages+tools → LLM.request)             ▼
                                              LLMClient.stream(LLMRequest)
                                                       │ compile(): resolveRequestOptions
                                                       │   → normalizeToolHistory → sanitize
                                                       │   → ToolSchemaProjection.tools
                                                       │   → applyCachePolicy/applyEffortUpdates
                                                       │ route.body.from(request)  ← 协议降级
                                                       │ route.prepareTransport()  ← endpoint+auth
                                                       ▼
                                              transport.execute → frames(SSE)
                                                       │ decodeEvent → protocol.stream.step 状态机
                                                       ▼
                                              Stream<LLMEvent>（text-delta/tool-call/finish/…）
                                                       │
     SessionStep.attempt（step.ts:107 llm.stream）←────┘  publisher 投影到 Bus + 工具执行
```

三层职责的取舍（见 `packages/ai/AGENTS.md`）：ai 包**只跑一次模型调用**，不知道 Session/权限/插件；Session 侧组装请求、消费事件；协议层只管"这个 API 长什么样"，不知道 URL/鉴权（那是 Route 的部署四轴）。各文件在管线中的角色：

| 层 | 文件 | 输入 | 输出 |
|---|---|---|---|
| 目录 | models-dev.ts | 外部 api.json / snapshot | `Snapshot{info, models, environment}[]` |
| 目录状态 | provider.ts / model.ts | Snapshot + config + 凭据 | `Provider.Snapshot` / `Model.Info` 索引 |
| 解析 | model-resolver.ts + variant.ts | `Info` + variant + credential | `Resolved{model: LanguageModel, ref, …}` |
| 请求组装 | session/model-request.ts | transcript + tools + system + agent | `LLMRequest`（经 hooks） |
| 执行 | llm.ts → route/client.ts | `LLMRequest` | `Stream<LLMEvent, AIError>` |
| 协议 | protocols/*.ts | 统一事件↔wire 双向 | body 构造 + 事件归一 |
| 消费 | session/runner/step.ts | `LLMEvent` 流 | Bus 投影 + 工具执行 + Outcome |

## 2. 模型目录（models-dev）

数据流：外部 `https://models.opencode.ai/api.json`（默认源，models-dev.ts:279）是权威上游；仓库内置 `packages/core/src/models-dev/snapshot.txt`（5.7 MB 单行 JSON）是**启动下限**——KV 缓存和网络都失败时保证目录非空（models-dev.ts:361-363 注释明言 "bundled snapshot is the boot-time floor"）。快照由 `bun run script/update-models-snapshot.ts` 提交刷新（models-dev.ts:281-283 注释）。

加载数据流（populate，models-dev.ts:385-401）：

```
populate()
  ├─ loadFromFile                     ← options.file 显式覆盖（测试/离线，354-359）
  ├─ loadFromCache                    ← KV "models-dev:catalog"（5.7MB 已超部分后端
  │     │                                2MB 值上限；写失败仅告警绝不致命，365-376）
  │     └─ 值损坏 → kv.remove 后继续
  ├─ bundledSnapshot                  ← decode+normalize 结果进程级 memo（287-297；
  │     │                                每 isolate 一次，见不变量 4）
  │     └─ options.snapshot===false 可禁用（363）
  ├─ !fetch → 返回 []
  └─ lock.withPermit（双检锁）        ← 再查一次缓存，miss 则 fetchApi+writeCache
```

`normalize()`（models-dev.ts:96-132）把上游 `SourceProvider/SourceModel` 转成 `Snapshot{info, models, environment}`：

| 上游字段 | 下游去向 | 说明 |
|---|---|---|
| `id/name/npm/env/api` | `Provider.Info` | npm 包名经 `AISDKNative.native()` 映射到 `@opencode/ai/providers/*`（models-dev.ts:85-94） |
| `reasoning_options` | `Variant.resolve(supports)` | effort/toggle/budget_tokens 展开成 variants（models-dev.ts:115，supports 在 196-202 过滤 null effort） |
| `experimental.modes` | 额外 mode 模型 | 每个模式生成 `<model>-<mode>` 新目录项，cost 与 base 合并（models-dev.ts:117-127，mergeCost 在 162-190） |
| `interleaved` | `Model.compatibility` | 字符串/`{field}` 归一为 `reasoningField`（model.ts:316-321） |
| `cost/tiers/cache_*` | `Model.Info.cost` | 数组：默认档 + 按 context size 分层档（models-dev.ts:139-160） |
| `limit/modalities/tool_call` | `capabilities`/`limit` | models-dev.ts:232-246 |

刷新：后台 fiber 每 5 分钟 `refresh()`（ttl=Duration.minutes(5)，models-dev.ts:330、431-434），双检锁（Semaphore 1）内：未过期且非 force 直接返回；拉取后 sha256 body digest 与存储一致则**跳过缓存写、invalidate 与 Refreshed 事件**（models-dev.ts:414-418）——models.dev 很少在两次轮询间变化，避免下游 provider/model 更新监听者无谓抖动；失败仅 logError + ignore，目录维持旧值。上游的 `canonical_model_id`（如 `alibaba/qwen3.6-flash`）经 `Provider.nativeSettings` 变成 canonical provider 指向（model-resolver.ts:227），使网关模型直连官方端点。

目录消费侧（Provider.Service → Model.Service）：

```
ModelsDev.get() ──→ Provider.Service（catalog transforms：config 覆盖/自定义模型/凭据门控）
                        │ Snapshot{records, available, providers}（provider.ts:309-311，442-473 缓存：
                        │ 凭据变化但 available 集不变则不算目录变化，466-470）
                        ▼
                    Model.Service（model.ts:107-109 初始索引 byProvider；Editor 供变换；
                        Model.Event.Updated 广播，244-253）
```

`environment` 字段（每个 provider 的 env 变量名清单，如 `ANTHROPIC_API_KEY`）供凭据子系统发现可用 provider。

## 3. Provider/Model 解析

`model.ts` 是 Location 级的目录状态：初始数据来自 `Provider.Snapshot.available`（model.ts:107-109），`State.Transformable` 提供可变 Editor 供 config/凭据变换，之后 `bus.publish(Model.Event.Updated)` 通知（model.ts:244-253）。

默认模型逻辑（model.ts:268-273）：

```
Model.default()
  ├─ data.defaultModel（用户/配置显式设定）且 enabled → 返回
  └─ 否则 value.available.find(supportsText)   ← 首个支持文本的可用模型
```

`available` 排序有一个人工锚点：全部免费 OpenCode 模型时 `preferBigPickle` 把 `big-pickle` 提到首位（model.ts:293-300）。`small(providerID)` 给摘要/标题选小模型，按 `gpt-luna → gemini-flash-lite → gemini-flash → claude-haiku` family 优先级（model.ts:274-285）。

**variant（thinking 档位）解析**：variant.ts:6-9 定义三档支持——`effort`（枚举值）、`toggle`（none/thinking）、`budget_tokens`（min/max）。`resolve()`（variant.ts:16-23）按 package 家族展开为 overlay 变体，产出 `[{id:"none",...off}, ...main]`（toggle 时 none 在前）：

| package 家族 | effort overlay | toggle overlay |
|---|---|---|
| Anthropic 系 | `thinking:{type:"enabled", budgetTokens}`（74 行 fitThinkingBudget） | 同左 + disabled |
| OpenAI Responses 系 | `reasoningEffort + reasoningSummary:"auto" + include:ENCRYPTED_REASONING`（98-104） | — |
| Gemini 系 | `thinkingLevel + thinkingSummaries:"auto"`（109-111） | — |
| openai-chat 系 | `settings.reasoningEffort`（78-79） | openaiChat 降级（85 行） |
| alibaba 系 | `reasoningEffort` + `body.thinking:{type:"enabled"}`，none 走 disabled（123-127） | body 模板 disabled/enabled（128-129） |
| nvidia 系 | — | nvidiaTemplate body toggle（129-130、140） |

Alibaba thinking budget 封顶 64k 而非模型 output limit（31 行注释）。effort 默认值集 `EFFORTS`，responses 家族扩展为 `["none","minimal",...EFFORTS,"xhigh"]`（95 行）。`ModelResolver.withVariant()`（model-resolver.ts:137-161）在解析时把 variant 的 `settings/headers/body` overlay 合并进 Info——**不改变已提交的目录快照**（注释 136 行），`variantID==="default"` 视为无变体（141 行）。

**运行时解析** `fromCatalogModel`（model-resolver.ts:196-260）主链：

```
fromCatalogModel(model, credential, deps)
  ├─ prepareRuntimeModel                 ← 剥掉空串 apiKey（262-265）
  ├─ 路线一：package 是 @ai-sdk/* 老包（isAISDK，provider.ts:25）
  │    → AISDK 桥接 loadAISDK + prepareProviderSettings（204-218）
  ├─ 路线二：@opencode/ai/providers/* 包
  │    ├─ packageName(specifier) 无值 → UnsupportedPackageError（219-220）
  │    ├─ prepareProviderSettings：baseURL 占位符 ${VAR} 环境变量展开（277-296）
  │    ├─ loadPackage（npm Service 动态 import，provider.ts:157）
  │    ├─ settings = mapped（无凭据时剥 accessToken/apiKey/authToken）
  │    │   + canonical provider + nativeCredentialSettings + headers/body（225-231）
  │    └─ module.model(modelID, settings) → LanguageModel.update(...)
  │         provider=canonical ?? providerID、compatibility 合并、
  │         timeouts/encoding 落到 route http defaults 而非包 settings（232-248，
  │         注释："Timeouts and body encoding are transport policy"）
  └─ 前置校验（168-194）：
       validateProviderVariables（${VAR} 全可解析，267-306）
       native compaction 策略必须配到有 compact 能力的 route（182-193）
```

随后两道校验：`validateProviderVariables` 检查 baseURL 里 `${VAR}` 占位符是否全部可解析（model-resolver.ts:267-306，失败抛 `UnresolvedProviderVariablesError`）；native compaction 策略在无 route compact 能力时**前置失败**（model-resolver.ts:182-193）。凭据形态映射：key→`apiKey`、oauth→token（anthropic 用 `authToken`、vertex 用 `accessToken`，model-resolver.ts:308-329）、bedrock 强制 sigv4（313）。`layer` 里的 `load`（model-resolver.ts:374-410）返回 `Resolved{model, ref, capabilities, cost, limit, compaction, transport, chunkTimeout}`——注意 `ref` 是目录身份，**durable 记录用 ref 而非 API model id**（110-114 注释）。enabled 但无凭据且包用 API key 鉴权时，降级为 `Auth.none` 让请求打到provider 拿 401（model-resolver.ts:389-395）。

`ModelResolver.resolve()` 无入参时：`models.default()` → 无 package 则退到 `available` 里首个 `hasPackage && supportsText`（model-resolver.ts:412-428）。

## 4. llm.stream 统一入口

`llm.ts` 是薄门面：`stream(input)` = `requestEffect`（把宽松 `RequestInput` 规范成 `LLMRequest` 类实例，llm.ts:57-84——system 归一 SystemPart、prompt 追加为 user 消息、tools 包 ToolEntry）→ `LLMClient.stream`（llm.ts:48-55）。`generate` 是 stream 的 fold：`Stream.runFold(LLMResponse.empty, LLMResponse.reduce)` 后 `LLMResponse.complete`，失败归一为 incomplete-stream（client.ts:610-616）。`generateObject` 强制合成工具调用而非 provider 原生 JSON mode，保证跨协议行为一致（llm.ts:156-167 注释；decode 失败也归 InvalidProviderOutput，142-152）。

`LLMRequest`（schema/messages.ts + options.ts）是全协议唯一请求形态：`{model, system: SystemPart[], messages: Message[], tools: ToolEntry[], toolChoice, generation, providerOptions, http, promptCacheKey}`。providerOptions 是开放 record（前向兼容原则），各协议在 `decodeOptions` 里挑自己认识的键。

真正的执行在 `route/client.ts`。`Route` 是五元组组合（client.ts:518-531 注释）：**Protocol（说什么）+ Endpoint（发到哪）+ Auth（怎么鉴权）+ Framing（怎么切帧）+ Transport（怎么传）**。`Route.make` 的四轴正交设计意味着新 provider = 组合既有协议 + 换 endpoint/auth，`route.with(patch)` 派生部署变体（client.ts:393-410）。`LLMClient.layer` 的 stream 全链（client.ts:602-608, 618-624）：

```
compile(request)                                   ← client.ts:574-587
  ├─ prepareRequest（client.ts:555-572）
  │   ├─ resolveRequestOptions：normalizeToolHistory + route/model/request 三级 merge
  │   │   （generation/providerOptions/http，client.ts:265-280）
  │   ├─ sanitizeSurrogates（剥 Unicode 代理对，某些 provider JSON 编码崩）
  │   ├─ 工具按 type:name 去重（工具与命名空间可同名，client.ts:558-562）
  │   ├─ ToolSchemaProjection.tools（§7）
  │   ├─ applyEffortUpdates（Message.effort 标记 → 请求参数，需协议声明 supportsEffortUpdates）
  │   └─ applyCachePolicy（缓存断点自动放置，cache-policy.ts）
  ├─ route.body.from(request)    ← 协议降级，产出 provider 原生 body（Schema 验证）
  └─ route.prepareTransport(body)（endpoint + auth + middleware → HttpPrepared）
streamPrepared（client.ts:423-501）
  ├─ transport.execute → execution{frames, http, body?, complete?}
  ├─ decodeEvent：frame → 协议事件（Schema 解码，失败包 eventError + 原始 wire body；
  │   frameError 保留原始帧做错误正文，client.ts:430-444）
  ├─ terminal 谓词存在则 Stream.takeUntil
  ├─ 状态机：state = protocol.stream.initial(request)
  │   每个 event → protocol.stream.step(state,event) → [next, LLMEvent[]]
  │   flatMap 展平回吐；onHalt 在帧流自然结束时 flush 补发（474-481）
  └─ requireTerminalEvent（client.ts:341-355）
      ├─ finish/providerError 后再吐事件 → 报错（防重复终态）
      └─ 流结束仍无 terminal 事件 → incomplete-stream AIError
```

统一 `LLMEvent` 联合（schema/events.ts:307-327）按 content-block 生命周期组织，共 19 个变体：

| 组 | 事件 | 说明 |
|---|---|---|
| 文本 | `text-start/delta/end` | end 带权威全量 text 可覆盖累积增量（156-158 注释） |
| 推理 | `reasoning-start/delta/end` | 同上结构；providerMetadata 携带签名等 |
| 工具 | `tool-input-start/delta/end/error` + `tool-call/tool-result/tool-error` | input-delta 带尽力解析的增量 JSON；tool-call 是定局事件（231-241） |
| 媒体 | `media` | 模型直接产出的一级媒体资产（293） |
| 终态 | `step-finish`（单轮）/ `finish`（终态） | FinishReasonDetails{normalized, raw, category?, explanation?}（270-278） |
| 错误 | `provider-error` | 流内错误走事件通道而非 Stream 失败（299-305） |
| 其它 | `compaction-part`、`step-start` | 服务端 compaction 块 / 多轮边界 |

`generate` 之外的第三入口 `compact`（client.ts:632-648）分 endpoint/trigger 两机制，仅 compact 能力路由可用（`canCompact` 类型守卫 236-248）；step.ts 不直接调用它，由 runner 侧 compaction 流程使用。

## 5. 协议层走读（逐协议）

### 5.0 四协议背景知识（外部视角速览）

> 来源：元宝"四种主流 LLM 网关协议对比"对话校验收录（2026-10-26）。以下从**协议本身**（而非仓内实现）的视角速览——四者是当前主流 LLM 网关的四套线协议契约，不是"同一模型四种叫法"。

| | openai-chat | open-responses | anthropic-messages | gemini |
|---|---|---|---|---|
| 端点 | `POST /v1/chat/completions` | `POST /v1/responses` | `POST /v1/messages` | `POST /v1beta/models/{model}:generateContent` |
| system 位置 | messages 里 `role:system` 一条 | `instructions` 或 system item | ★顶层 `system` 字段（不进 messages） | 顶层 `systemInstruction.parts` |
| 内容单元 | 扁平 `content:string` | typed items（input_text/function_call/reasoning） | content blocks（text/tool_use/tool_result/thinking） | parts[]（text/functionCall/inline_data） |
| 工具结果回传 | `role:tool` + tool_call_id | function_call_output item + call_id | user 消息嵌 tool_result block（关联 tool_use_id） | functionResponse part（按函数名） |
| 流式事件 | SSE `choices[].delta`（[DONE] 哨兵收尾） | 结构化事件（response.created/item.*/delta） | message_start/content_block_delta/message_delta | candidate chunk 分片（结构最平） |
| thinking 保真 | 无一等字段，易丢 | ★原生 reasoning item（最好） | thinking block + 签名（须原样回放） | thought + thoughtSignature（特殊映射） |

三条与仓内实现的印证/补充：

1. **"thinking 保真度"正是仓内语义对齐的难点**——anthropic 的签名回放、gemini 的 thoughtSignature 映射（§5.4），与上表"各家各有形态"一致。
2. **选型原则："先看客户端/上游发什么，别看模型叫什么"**——这正是仓内"协议由目录 package 字段决定、与 provider 名无关"的外部成因（同一个模型多家网关四种入口都能调，转换层决定损耗在哪）。
3. **网关通用做法印证了本仓架构**：暴露多入口 → 内部一套中间表示（IR）→ 请求侧转上游格式、响应侧转回调用方协议——opencode 的 `LLMRequest/LLMEvent` 就是那个 IR，§0.5 的"双向 Adapter"即此模式。元宝提到的"全转 Chat Completions 能跑 80% 但丢 thinking 细节"，正是 opencode 保留四套协议而不收敛为一套的理由。

`Protocol` 接口（route/protocol.ts:39-53）= `body{schema, from}` + `stream{event, initial, step, terminal?, onHalt?}`。每个协议实现都是同一个形状：Schema 定义 wire 结构 → `fromRequest` 降级函数 → SSE/事件流状态机 → `Protocol.make` + `Route.make`。协议被选择的方式：**provider 部署文件组合 Route，目录数据的 package 字段决定加载哪个文件**；如 `providers/deepseek.ts:23-29` 直接 `protocol: OpenAIChat.protocol` 复用 chat 协议——DeepSeek/Together/Cerebras 等无需各抄 300 行（protocol.ts:34-36 注释）。open-responses 是"provider 中立 Responses 基线"，Bedrock Mantle 与 Vertex Responses 复用它（providers/amazon-bedrock-mantle.ts:48-50）。

### 5.1 anthropic-messages（1,676 行）

wire 结构（Schema 分层定义）：请求体 `AnthropicMessagesBody`（394 行）由 `AnthropicBodyFields` 组合；内容块是 tagged union——`text`（带 `cache_control: {type:"ephemeral", ttl}`，82-88）、`image`（base64/url/file 三种 source，101-116）、`thinking`、`tool_use`、`tool_result`、`server_tool_use`、`redacted_thinking`、`compaction`（服务端上下文管理块）。usage 结构 `AnthropicUsage` 带 `iterations` 数组（多轮 server tool 循环的逐轮计量，404-419）。`ContextManagement`（62-76）声明服务端 compaction：`compact_20260112` edit + `input_tokens >= 50000` 触发器。

请求侧 `fromRequest`（1038-1100+）逐项：

```
fromRequest(request)
  ├─ decodeOptions(providerOptions)          ← 驼峰/蛇形双拼写（342 注释）
  ├─ resolveEffortUpdates(request, effort)   ← effort 动态改档
  ├─ Cache.newBreakpoints(4)                 ← 断点预算上限 4（1050）
  ├─ flattenToolRequest → lowerTool          ← 每个工具可带独立断点
  ├─ system：非空 part 各自挂 cache_control
  ├─ lowerMessages                           ← 消息尾断点
  ├─ maxTokens = generation?.maxTokens ?? 32_000
  ├─ thinking = fitThinking(options.thinking, maxTokens)   ← 预算 ≤ maxTokens，最低 1,024
  └─ context_management（可选，服务端 compaction 策略）
```

**4 断点预算按失效序分配 tools→system→messages**，超额丢 message-tail 断点并告警（1050-1052、1064-1067）；`tool_choice` 在无 tools 时整体省略（Anthropic 拒绝无 tools 的 tool_choice，1054）；beta 头由 transport 包装器动态追加：恒定 `interleaved-thinking-2025-05-14`，compaction 场景（请求或回放 compaction 块）加 `compact-2026-01-12`，block_binding thinking 加 THINKING_BINDING_BETA（1638-1652，`requiredBetaHeaders`）。

事件侧状态机（step 函数 1545-1583 附近分发）：SSE 事件集 `message_start/content_block_start/content_block_delta/content_block_stop/message_delta/message_stop/ping/error`（41-53，`Framing.sseEvents` 切帧）。state 字段（initial，1603-1610）：`ToolStream<number>`（按 block index 累积工具参数 JSON 增量）、`reasoningSignatures`（thinking 签名，回放时附回）、`Lifecycle`（start/delta/end 配对检查）、`compactions`。要点：

- `redacted_thinking` 与 `server_tool_use` 的 tool result **整块到达无 delta 序列**——payload 在 `content_block_start` 的 `data`/`content` 字段（432-437 注释）
- `message_delta` 携带 `stop_reason/stop_details`（460-466），归一为 FinishReason
- `content_block_delta` 的 `delta.type` 不在已知集则静默吞掉（前向兼容，1555-1557）
- route：`?beta=true` 仅 anthropic 直连 provider 时追加（1668-1672）；恒定头 `anthropic-version: 2023-06-01`（1674）

### 5.2 openai-chat（1,322 行）

最厚的"兼容层"协议——同一协议要伺候 OpenAI、DeepSeek、Together、Groq、OpenRouter、硅基流动等所有 chat/completions 形端点。降级函数按 `model.compatibility` 与 baseURL 启发式双重分叉：

```
fromRequest(request)                                  ← 811-856
  ├─ reasoningField 冲突检查（RESERVED_REASONING_FIELDS，813-816）
  ├─ flattenToolRequest
  ├─ 探测/读取 compatibility 开关：
  │   maxTokensField（max_tokens vs max_completion_tokens）
  │   supportsStore / supportsUsageInStreaming / supportsStrictMode / zaiToolStream
  │   （818-831：compatibility 显式值 ?? provider+baseURL 探测）
  ├─ lowerMessages（reasoning 双方言：OpenRouter reasoning_details vs
  │   reasoning_content，80-107 注释；state 记 reasoningTextObserved 区分）
  ├─ tools：lowerTool（strict mode 可选）；有历史无活跃工具时发空数组
  │   而非 undefined（837-842，保持工具 schema 历史一致性）
  ├─ stream_options.include_usage（按支持度）
  └─ lowerOptions：prompt_cache_key 截 64 字符（shared.ts:41）、reasoning_effort
```

事件侧状态机 state（initial，1283-1300）：`ToolStream<number>`、`pendingTools`、`reasoningField/reasoningTextObserved/reasoningDetails/reasoningDetailsObserved/reasoningEmitted`（reasoning 归一化五件套）、`requireFinishReason`（compatibility 可关，默认 true）。终态用哨兵字符串 `[DONE]`：`step: event===DONE → []`、`terminal: event===DONE`（1292-1295）；`onHalt: finishEvents` 在帧流结束时兜底补发 finish（1297）。`mapFinishReason` 把各家 stop 原因（length/tool_calls/content_filter/error/…）归一到统一 FinishReason。tool_choice 经 `ProviderShared.matchToolChoice` 归一 auto/none/required/named 四态（345-347）。

### 5.3 open-responses（1,509 行）

provider 中立 Responses 基线（`PATH = "/responses"`，34 行）——不是 OpenAI 专有：Bedrock Mantle（providers/amazon-bedrock-mantle.ts:48-50）与 Google Vertex Responses（providers/google-vertex-responses.ts:32-33）都直接复用它。数据模型是 **input/output items** 而非 messages 数组：请求把 transcript 降级为 `input` 项列表（`input_text/input_image/input_file/input_video`，38-59），assistant 侧历史降为 `output` items 回放。

状态机 state（initial，1485-1491）：`outputItems`（按 item id 聚合 output_text/function_call/reasoning item）、`reasoningItems`（加密 reasoning 回放）、`tools: ToolStream<string>`（Responses 的工具参数 key 是字符串 id 而非数字 index——与 anthropic/openai-chat 的显著差异）、`message`。`step` 先经 `normalize(state, event)` 再喂内部状态机（1503）。

关键差异点：

- **服务端会话状态**：`previous_response_id` 续链与 checkpoint/continuation 体系（配套 `open-responses-continuation.ts` 208 行、`utils/responses-checkpoint.ts` 118 行、`utils/responses-compaction.ts` 200 行）——compaction 可以在服务端做，与前述 anthropic 的 `context_management` 是两条不同的服务端压缩路线
- **terminal 显式谓词**（1504）而非 `[DONE]` 哨兵；流式事件 `response.output_text.delta` 等按 item 组织
- 终态 recovery 不能在已吐内容后插 checkpoint（1330-1334 报错路径）
- transport 用 `HttpTransport.sseJson`（1511）

### 5.4 gemini（976 行）

请求侧：模型 id **嵌在 URL path**（`/models/<id>:streamGenerateContent?alt=sse`，967-970，SSE framing 在 URL 层 pin 死）；工具 schema 走 `parametersJsonSchema` 标准 JSON Schema（284，`functionDeclarations` 数组 145）；协议自带 `sanitizer: "gemini"` 使 Gemini schema 规则成为该 API 默认（947 附近，protocol 定义见 940-961）。

事件侧要点：

- **thought signature**——Gemini 3 回放 function call 必须带签名，否则 API 拒收；`requiresThoughtSignatureFallback` 按模型代际判定（43-47：gemini-1/2 与 gemini-pro 不需要），配 `skip_thought_signature_validator` 开关（35）
- 内容 part 是 camelCase（`inlineData/functionCall/thoughtSignature`，75-99），与其它协议的 snake_case wire 不同
- `NextRequest` 二进制帧与 JSON 事件 Union（947）——同一协议可接非 SSE 通道
- `onHalt: finish(state)` 兜底补发 finish（955）
- gemini-2.5 走 legacy tool media 路由（52，`routesLegacyToolMedia`）

其余协议是同一模式的家庭成员，均按"转换要点+关键差异"掌握即可：alibaba-chat/messages/responses（Qwen 系三皮肤）、meta-messages/responses、xai-responses、mistral-chat（工具调用 id 需修复，utils/mistral-tool-id.ts 34 行）、cohere-chat（334 行）、venice-chat（374 行）、zai-chat/messages、bedrock-converse（895 行，AWS SigV4 签名 + 二进制 event-stream framing，utils/bedrock-auth.ts 147 行 + bedrock-event-stream.ts）。media 类协议（openai-images/speech/transcription、google-*、fal/replicate/stability/bfl/runway 等）遵循同一 Protocol 形状但走 image/speech/video 客户端而非 LLMClient，本篇不展开。

## 6. 与 Session 的接缝

Session 侧组装点在 `session/model-request.ts`，消费点在 `session/runner/step.ts`（Step 调度时序归 ../5_timeline/5，此处只写接缝映射）。

入参映射（model-request.ts:110-118 + 279-296）：

| Session 侧输入 | LLM.request 字段 | 中间处理 |
|---|---|---|
| agent.system + 环境上下文 | `system: SystemPart[]` | SessionSystemPrompt 组装 |
| 会话 transcript（durable 消息） | `messages` | `toLLMMessages()` 转协议中立 Message；`unsupportedParts` 剥模型不支持的模态；`boundImages` 超预算时替换图片为占位文本（44-46） |
| Tool 注册表 + agent 工具集 | `tools: ToolEntry[]` | plugin hook `session.context` 可改名/改 schema，按对象身份回配（267-276） |
| 模型选择 | `model: model.model` | 来自 `SessionRunnerModel.Resolved`（runner/model.ts 包装 ModelResolver） |
| 输出预算 | `generation.maxTokens` | `outputLimit()` 按 catalog limit 与已用 input tokens 收缩（94-97） |
| 会话亲和 | `http.headers` + `promptCacheKey` | x-opencode-session-id 等 + `ses_` 前缀剥离（287-293） |

hook 接缝：`session.model.request` 可改 baseURL/headers → `route.with({endpoint})` 派生新 route（305-320）；改路由后 `SessionProviderContext.compatible` 校验 opaque 窗口不得跨部署——已安装路由 hook 不能把既有的不透明 provider 窗口发给另一个部署（323-331 注释）。`session.http.request` hook 还能拿到最终 HttpClientRequest（341-345）。

消费接缝：`SessionStep.attempt`（step.ts:69-）取 `LLMClient.Service`，`llm.stream(input.prepared.request, input.prepared.options)`（step.ts:107）后 `Stream.runForEach` 逐事件处理：

```
providerStream 逐事件（step.ts:108-130）
  ├─ 已有 overflowFailure 或 providerError → 跳过（幂等保护）
  ├─ publisher 未发布过 Step.Started → 先等 pendingStartSnapshot fiber
  │    （快照与请求并行起飞：快照只需在本地工具运行前就绪，74-75 注释）
  ├─ provider-error 且 isContextOverflowFailure 且 !outputStarted
  │     → 截获存 overflowFailure，不发布（112-118）
  ├─ publisher.publish(event) → Bus 投影
  └─ tool-call 且非 providerExecuted
        → fork 工具 fiber（uninterruptibleMask 内执行 → truncate →
           toolExecution/failTool 发布，120-130）
```

provider 流与工具执行**并行且各自保序**（"Provider and tool fibers retain per-source order without a shared writer queue"，step.ts:101 注释）。流结束后收敛：overflow 存在则压缩前告警（160-166）；`record.finish === "length"` → `Outcome.OutputLimit`（282）；工具拒绝/中断各有归类（270-280）。核心入口只接两个参数——`prepared.request`（组装完成的 LLMRequest）与 `prepared.options`（StreamOptions 传输中间件），Session 与协议层之间没有任何其它耦合通道。

## 7. 工具 schema 转换

内部 `Tool`（ai/src/tool.ts 的 `make`）持 Effect Schema/Zod/JSON Schema 三源，`toDefinitions` 输出 `ToolDefinition{inputSchema: JsonSchema}`。Session 侧的注册表 Tool（core/src/tool.ts，见 [9_tool.md](9_tool.md)）在 model-request 组装时已转成 `ToolEntry`，进入 ai 包后还有一道**统一投影**：`ToolSchemaProjection.tools`（tool-schema.ts:99-104）在 `prepareRequest` 里对每个请求跑一次（client.ts:565），命名空间嵌套递归（`tool.type !== "tool"` 的分支递归 `tool.tools`）。

```
ToolEntry.inputSchema（Effect 序列化的 JSON Schema）
  └─ objectRoot（tool-schema.ts:56-75）      ← 无条件先做
       ├─ 剥 Effect 空 struct 标记：not:{type:"null"} / anyOf 含裸 object 分支
       └─ 根强制 type:"object"（多数 provider 拒绝 {} 或裸 properties 根）
  └─ sanitizer 三级选择（81-96）：
       1. model.compatibility.sanitizer    ← 目录/配置显式（"none" 可关）
       2. model.route.sanitizer            ← 协议默认（gemini 协议恒 "gemini"）
       3. MODEL_NAMES 模型名正则            ← /gemini/i→gemini、/kimi/i→moonshot：
                                             网关与 OpenAI-compatible 端点后面的
                                             模型也拿到家族规则
       → gemini: GeminiJsonSchema.normalize / moonshot: moonshot() / 其余: root 原样
```

两家净化器要点：

| 净化器 | 规则 | 位置 |
|---|---|---|
| **Gemini** | `required` 只能引用已声明属性；Draft-04 布尔 exclusiveMin/Max 转数值形；递归进 properties/$defs/patternProperties 等 schema map；const/default/enum 等值集原样保留 | gemini-json-schema.ts:5-77，注释明言"只改被拒形状，其余原样发" |
| **Moonshot** | `enum` 必须有兄弟 `type`（从枚举值推断单类型或"类型+null"，13-20）；`prefixItems` 折叠为 `items`（数组态映射 anyOf，5-10）；丢 `unevaluatedItems` | tool-schema.ts:12-44 |

之后各协议 `lowerTool` 再降到 wire 形态：Anthropic `input` + `cache_control` 断点、OpenAI `parameters` + strict mode、Gemini `parametersJsonSchema`、Responses `parameters`（hosted tools 由 `responses-hosted-tools.ts` 决定）。缓存断点的自动放置策略在 `cache-policy.ts`：默认 "auto" 在最后一个工具定义与对话尾放断点，且按 provider 分叉——OpenRouter/网关、Gemini（只用最后断点，对话尾断点每步写新缓存反而更贵）、Qwen 各有专属策略（cache-policy.ts:58-60 注释）。

## 8. 错误与重试信号

`provider-error.ts` 两段式：**分类**（affirmative evidence 原则）+ **可重试判定**。

`classifyProviderFailure`（235-305）按优先级分类，命中即返回（code 集合在 120-178 行定义，正文提取在 195-211）：

```
classifyProviderFailure(input)
  ├─ clientScoped(4xx) + context_length_exceeded 系 code / 30+ 条正则
  │    → InvalidRequest{classification:"context-overflow"}
  │      模式表覆盖各家话术：OpenAI/Cloudflare 413/Bedrock 中途 validationException/
  │      DeepInfra/Novita/HF-TGI/Z.ai "Prompt 超长"（16-54）；排除节流与
  │      Cohere max_tokens 误导项（58-64）
  ├─ 413 / payload 正则 → InvalidRequest{payload-too-large}
  ├─ CONTENT_POLICY_CODES / 文案 → ContentPolicy
  ├─ 402 / QUOTA_CODES / 429+QUOTA_TEXT → QuotaExceeded
  │      （Zen 的 429/402 类型化账户上限不是节流，119-127 注释）
  ├─ 401 / 403 / AUTH_CODES / 400+AUTH_TEXT → Authentication
  │      （Google 400 INVALID_ARGUMENT 带 details.reason 也算，128-130）
  ├─ 429 / rate_limit 系 code / RATE_LIMIT_TEXT → RateLimit{retryAfterMs, rateLimit}
  ├─ 408 / 409 / 5xx /（无矛盾状态时）SERVER_CODES 或服务端话术 → ProviderInternal
  │      （4xx 带服务端 code 仍是 InvalidRequest——网关会替换不认识的 code，290-293 注释）
  ├─ INVALID_REQUEST_CODES → InvalidRequest
  ├─ 其余任何 4xx → InvalidRequest（确定性拒绝）
  └─ 兜底 → UnknownProvider
```

code 提取 `providerCodes`（307-335）扫 8 层嵌套布局：顶层 code/error.code/error.type、Google `details[].reason`、Azure `innererror.code`、OpenRouter `error_type`、Stability `errors[].name`、Zen 网关改写消息里的 `[code]` 标签（163）。

`isRetryable`（81-116）：

| reason | 可重试 | 备注 |
|---|---|---|
| RateLimit / ProviderInternal / UnknownProvider | ✔ | UnknownProvider 重试的理由：确定性失败几乎总带状态码或已知 code，认不出的恰是瞬态（98-101 注释） |
| InvalidProviderOutput | 仅 `incomplete-stream` | 流中断检测见 §10 |
| Transport | delivery≠rejected 且（非 accepted 或 operation=read） | WebSocket accepted 写已交付，读失败可恢复（89-95） |
| Authentication / QuotaExceeded / ContentPolicy / InvalidRequest / Timeout 等 | ✘ | |
| （任何）`x-should-retry` 头 | 头值优先 | 82-84 |

core 侧 `session/runner/retry.ts` 引用而不展开：`isRetryable` re-export（retry.ts:14），调度为指数 2s 封顶 10s × 最多 10 次、jitter、`retry-after` 上限 15 分钟（36-56），timeout 类单独限 3 次（62-64）。

## 9. 不变量与设计取舍

对照 `specs/v2/provider-policy.md`（先读，379 行）：**provider 配置与 provider 策略分离**——`providers` 描述端点/选项/模型覆盖，`experimental.policies` 决定"用不用"；配置正确+凭据有效仍可被策略 deny（spec 开篇）。`provider.use` 资源是 provider ID，通配符匹配、**后写的匹配语句胜出**（`findLast` 语义）；策略语句跨文档**逆序**读（用户全局覆盖仓库），文档内保持书写序。策略在目录全变换之后、由终端插件评估——所以本层代码里看不到策略检查，它切在 catalog 的 `available` 门外。

代码内可验证的不变量：

1. **一次 stream 调用恰终态一次**：`requireTerminalEvent`（client.ts:341-355）双检——终态后再吐事件报错、无终态结束报 incomplete-stream。
2. **目录身份与 API 身份分离**：`Resolved.ref` 用于 durable 记录与展示，`model.id` 才是 API model id（model-resolver.ts:110-114）。
3. **variant overlay 不改快照**：withVariant 每次产新 Info（model-resolver.ts:136-137 注释）。
4. **normalize 每 isolate 一次**：5.7MB 快照是模块级常量，多 runtime 共存（Cloudflare DO 同 isolate）不重复解码（models-dev.ts:281-287 注释）。
5. **工具 schema 投影在协议降级前统一做**（tool-schema.ts:98 注释），协议实现不各自净化。
6. **cache 断点预算按失效序分配**（anthropic-messages.ts:1050-1052 注释），over-mark 时保 tool 断点丢 message-tail。
7. **协议 ≠ 部署**：Protocol 无 URL/鉴权知识，Route 四轴组合（protocol.ts:24-33 注释）——这是 66 个 provider 文件平均 <100 行的原因。
8. **generateObject 强制合成工具**而非原生 JSON mode，保证跨协议一致（llm.ts:156-167 注释）。
9. **快照是下限不是权威**：定时刷新仍在快照之上进行（models-dev.ts:361-362 注释）。
10. **错误分类记录肯定性证据**：确定性失败需要正面识别（4xx 状态/配额/鉴权/策略信号）；认不出的留 UnknownProvider 且**可重试**——瞬态失败恰以分类器预料不到的形态到达（provider-error.ts:229-234 注释）。
11. **协议对未知事件前向兼容**：anthropic 未知 `delta.type` 静默吞掉、providerOptions 接受未知键透传——provider 高频上新，闭字面量只用于 opencode 分支依赖的值（packages/ai/AGENTS.md 前向兼容原则）。
12. **decode 一次在边界**：LLMRequest 宽松输入在 `request()` 规范化一次，之后全链传类型化值；协议 body 由 `body.schema` 在 encode 前验一次。

取舍记录：KV 写失败不致命（DO SQLite 2MB 上限已超，365-368 注释）；`packages/llm/` 目录已退役为空壳（仅 node_modules），LLM 职责整体并入 `packages/ai`；`compileRequest` 是测试专用投影、不进包 barrel（client.ts:589 注释）。

## 10. 失败与恢复

- **不完整流检测**：三处独立把关——(a) client.ts `requireTerminalEvent` 流尾无终态 → `InvalidProviderOutput{classification:"incomplete-stream"}`（332-339）；(b) `generate` fold 后 `LLMResponse.complete` 失败同因（610-616）；(c) step.ts 里 finish reason 为 `unknown` 也包装成 incomplete-stream（step.ts:172-180，"provider response ended with an unknown finish reason"）。
- **流中断后的分类恢复**（step.ts:184-196）：`Transport` 失败带 `recovery: retry-full/rotate-and-retry-full` 且尚无 durable 输出 → `Outcome.RecoverFull`（一次性全量重建，消耗 runner 的 recoverContinuation 名额）；`isInterruptedStream`（step.ts:291-296）= incomplete-stream 或 Transport read 失败，即便 isRetryable 为 false 也允许已开输出后的续写重试（192-194——重试与 Continue 由 runner 的 Outcome 分派，时序归 ../5_timeline/5）。
- **context overflow 恢复**：流内 `provider-error` 且分类 context-overflow 且 `!outputStarted` → 截获不发布，交 runner 触发压缩（`Outcome.Compacted`，step.ts:112-118、160-166）；已经开始输出后才到的 overflow 只能正常发布走失败路径。
- **unknown finish reason**：见上 (c)，防止 provider 静默截断被当成功。
- **终态后事件**：terminal 事件后 provider 再吐任何事件 → eventError（client.ts:345-349），防协议状态机错乱污染已完整的响应。
- **目录刷新失败**：refresh 的 cause 只 logError + ignore（models-dev.ts:425-428），目录维持旧值；缓存写失败同策略（365-376）。
- **流内错误双通道**：协议事件层的 `error`（如 anthropic onError）归一为 `provider-error` 事件；传输/解码层失败走 Stream 失败通道由 retry 策略接管——step.ts 对两者统一收敛（181 行 llmFailure 选择逻辑：stream 失败优先，否则 unknownFinish 兜底）。
- **鉴权缺失的运行时探测**：enabled 无凭据的 API-key 包走 `Auth.none` 放行（model-resolver.ts:389-395）——让 401 本身成为"教用户配 key"的信号，而非启动期硬失败。

## 演进注记

旧库笔记 4.2《provider-model》只覆盖 `core/src/provider.ts`、`model.ts`、`catalog.ts` 三文件——那时代码用 Vercel AI SDK 的 `@ai-sdk/*` 包直接驱动模型，opencode 自己没有协议层。现码的三大变化：

1. **自建 ai 包**（`packages/ai`，~20k 行）：Route/Protocol/Transport 分层取代 AI SDK 的 provider 抽象；`catalog.ts` 已删除，目录职责拆进 `models-dev.ts`（外部数据）+ `provider.ts`/`model.ts`（目录状态）；`@ai-sdk/*` 只剩 `aisdk-native.ts` 桥接兼容。
2. **协议层从无到有**：46 个协议文件覆盖 anthropic/openai-chat/open-responses/gemini/bedrock 等家族，加 media（image/speech/transcription/video）协议；统一 LLMEvent 事件模型（19 变体 content-block 生命周期）替代 AI SDK 的 streamPart。
3. **目录数据外置与策略化**：旧笔记时代的模型列表硬编码在仓库（catalog.ts）；现由 models.dev 外部源 + bundled snapshot + KV 缓存三层供给，variant（thinking 档位）从配置概念变成目录数据驱动的 overlay 体系，且新增 provider-policy 策略层（`provider.use` 准入门控，specs/v2/provider-policy.md）与 canonical_model_id 网关直连重写。

`packages/llm/` 是退役残留（目录仅 node_modules，无 package.json/src），本篇不为其立节。另：旧笔记时代模型列表硬编码于 `catalog.ts`；现码的目录数据链是 models.dev 外部源 → snapshot 兜底 → KV 缓存 → Provider/Model 两级目录状态，variant（thinking 档位）从配置概念变成目录数据驱动的 overlay 体系，且新增了 provider-policy 策略层（`provider.use` 准入门控）与 canonical_model_id 网关直连重写。

## 相关篇

- [4_session.md](4_session.md)：Session 聚合——本篇 §6 接缝的对侧
- [../5_timeline/5_prompt-chain.md](../5_timeline/5_prompt-chain.md)：Step 调用 llm.stream 的完整时序（本篇只写接缝）
- [9_tool.md](9_tool.md)：内部 Tool 定义体系（本篇 §7 的输入侧）
- [1_data-layer.md](1_data-layer.md)：KV/State 基础设施（models-dev 缓存依赖它）
- [0_effect-infra.md](0_effect-infra.md)：Stream/Layer 心智（本篇大量使用）
