# 可观测性（Observability）

> 一句话：**观测三件套——本地 `fileLogger`（单文件追加 + 原地压缩）打底、OTel（OTLP logs/traces，env 开关，默认关）为可选升级、durable 事件日志为"天然遥测源"；外加桌面启动 marks 计时与 TUI `OPENCODE_DIRECT_TRACE` 磁道两块专项。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-22），行号均为当日实测。
> 权威出处：`packages/util/src/observability/`（logging.ts 179 行 / otlp.ts 109 行 / shared.ts 8 行）、`packages/util/src/observability.ts`（组装层）、`packages/server/src/request-tracing.ts`、`packages/desktop/src/main/lifecycle/marks.ts`、`packages/cli/src/mini-host.ts`。
> [返回索引](README.md) | 互链：日志**位置**见 [../3_ops/1_log-locations.md](../3_ops/1_log-locations.md)（本篇只写机制）；OTel/Honeycomb **是什么**见 [../2_tech-stack/2.4_infrastructure/7_opentelemetry.md](../2_tech-stack/2.4_infrastructure/7_opentelemetry.md) / [8_honeycomb.md](../2_tech-stack/2.4_infrastructure/8_honeycomb.md)（本篇写仓库怎么用）。

---

## 代码地图

```
packages/
├─ util/src/
│  ├─ observability.ts            # 组装层：fileLogger+OTLP → Effect Logger Layer；失败回退纯本地
│  └─ observability/
│     ├─ logging.ts               # fileLogger / trim 原地压缩 / 级别 / stderr 开关
│     ├─ otlp.ts                  # OTLP logs + traces 导出（BatchSpanProcessor）
│     └─ shared.ts                # runID()：进程级 8 位随机 ID（workerd 懒生成）
├─ core/src/
│  ├─ bus.ts                      # durable 事件总线：log() 游标跟随读（事件即遥测）
│  └─ event-logger.ts             # 把 5 类 updated 事件转成文本日志
├─ server/src/
│  ├─ request-tracing.ts          # withoutParentSpan：请求不继承启动 span，但续接入站 traceparent
│  └─ event-feed.ts               # SSE 事件订阅（4096 容量背压）
├─ desktop/
│  ├─ src/main/lifecycle/marks.ts # 启动计时点（entry/ready/window/served/…）
│  ├─ src/main/native/logging.ts  # marks.logging + "app starting" 日志（带全量 marks）
│  └─ scripts/bench-startup.ts    # 冷启动基准：消费 marks，输出阶段中位数
└─ cli/src/mini-host.ts           # createTrace：OPENCODE_DIRECT_TRACE → JSONL 磁道
```

---

## 0. 恐惧清单

| 恐惧 | 靠什么 | 落点 |
|---|---|---|
| 用户报障，"当时到底发生了什么" | 本地日志（每行带 run ID 可分进程）+ durable 事件表可全量重放 | §1 / §2 |
| 生产环境黑盒，想集中收日志/trace | OTLP 导出（endpoint 配了才开） | §3 |
| 跨进程请求链路断了，谁调谁 | HTTP trace 传播（traceparent 续接 + 请求不挂启动 span 下） | §4 |
| 启动慢，慢在哪一段 | 桌面 marks 计时点 + bench-startup 阶段拆分 | §5 |
| TUI 交互会话无法离线复盘 | direct trace JSONL 磁道（send.prompt/recv.event/ui.patch…） | §6 |
| 日志盘满 | 原地压缩：>50MB 砍头保 ~25MB，1 小时一查 | §1.3 |
| OTel 端点不可达 | Layer 失败自动回退纯本地日志（有测试锁定） | §8 |
| provider 调用花多少钱 | 不是日志的活：durable 事件里的 `session.stats`（tokens/cost/工具 P50） | §2 |

设计基调：**默认零外发**（无 endpoint 就没有 OTel Layer、没有 Sentry），一切观测数据先落本机；上集中观测是部署者的显式选择。

---

## 1. 日志体系（fileLogger 机制）

> 位置与两套体系对比归 [3_ops/1](../3_ops/1_log-locations.md)；本篇讲实现机制。

### 1.1 进程身份：runID

`packages/util/src/observability/shared.ts:5-7`：

```ts
export function runID(): string {
  generated ??= crypto.randomUUID().slice(0, 8)
  return generated
}
```

全机共享一个日志文件、多个 opencode 进程并发追加，靠**每行带 8 位 run ID**区分来源（logging.ts:22 `["run", id]`）。懒生成是因为 workerd 禁止在全局作用域取随机值（shared.ts:1-2 注释）——同一约束也出现在组装层：`observability.ts:52-53` 注释说明 `Layer.suspend` 推迟模块作用域的文件 I/O。

### 1.2 结构化平面格式

`logging.ts:16-31` 的 `formatter` 把 Effect 结构化日志拍平成 `key=value` 单行：

- 固定前缀 `timestamp=… level=… run=…`，随后是 message、cause、spans、annotations 逐个展开；
- `flatten()`（33-46 行）递归展开嵌套纯对象为点路径（`a.b.c=…`），`WeakSet` 防循环引用输出 `[Circular]`（38 行）；
- `format()`（54-57 行）：含空白/`=`/引号/反斜杠的值 JSON.stringify，否则裸写。

有测试锁定这两个行为：`core/test/effect/observability.test.ts:87`（"concurrent runs with a run on every line"）与 `:115`（"flattens nested objects"）。

### 1.3 原地压缩（trim）——本篇最精巧的一段

`logging.ts:6-14` 常量：`LOG_MAX_BYTES=50MB`、`LOG_KEEP_BYTES=25MB`、每小时查一次（`LOG_TRIM_INTERVAL`），锁过期 5 分钟视为死进程。

为什么**原地**而不是轮转？79-85 行注释原文意图：其他进程以 `O_APPEND` 持有同一文件，改名轮转会让他们继续写已被卸载的旧 inode、输出全丢；原地砍头则追加者无感。代价是最后一次 read 与 truncate 之间别进程的 flush 会丢——几毫秒窗口、每小时一次，"accepted trade for not wrapping every log write in a cross-process lock"。

机制（86-124 行）：

1. **mkdir 锁**（98-108 行）：`${target}.trim` 目录做跨进程互斥——mkdir 在所有平台原子；`AlreadyExists` 则查锁 mtime，超 5 分钟判死锁删之并本轮让路（`breakStaleLock`，133-142 行）；
2. stat 超过 max 才动手（110-111 行）；
3. `lineStart`（145-157 行）从 `size - keep` 向后找第一个换行，保证砍在行边界；
4. 64KB 分块把尾部**前移覆盖到文件头**（写游标永远落后读游标，117-122 行），最后 `truncate(written)`（123 行）。

注释还写明锁竞态的边界情形（129-132 行：两个进程同见死锁的极端时序）及升级路径（"see EffectFlock if it is"）——防御到"够用"为止。

### 1.4 级别与开关

- `minimumLogLevel()`（logging.ts:164-173）：`OPENCODE_LOG_LEVEL` ∈ DEBUG/INFO/WARN/ERROR，默认 INFO；
- stderr 旁路（159-162 行）：`OPENCODE_PRINT_LOGS=1` 时同格式写 stderr——终端调试用，不落盘不压缩；
- `loggers()`（175-177 行）= fileLogger + stderrLogger 两个 Logger。

### 1.5 组装与装配点

`util/src/observability.ts` 的 `layer()`：本地 fileLogger Layer 与"fileLogger+OTLP"Layer 二选一，OTLP 初始化 `Layer.catchCause(() => local)`（50 行）兜底回退。CLI 主入口 `packages/cli/src/index.ts:129` 用 `OTEL_EXPORTER_OTLP_ENDPOINT/HEADERS` + `OPENCODE_CLIENT/VERSION/CHANNEL` 装配。core 侧另有 `core/test/logging.test.ts:7/11` 锁定 local/non-local 两种日志文件名。

### 1.6 event-logger.ts 的角色

`core/src/event-logger.ts` 全文 17 行：订阅 bus，把 `agent/provider/model/command/config.updated` 这 5 类**目录变更事件**转成 `Effect.logInfo("event", { event })`——即"配置发生了什么变化"进入文本日志。它是事件→文本日志的**单向小桥**，不是事件存储（那是 bus 的 event 表）。

---

## 2. 事件即遥测（durable 事件作为观测源）

opencode 的核心观测源其实不是日志，是 **durable 事件本身**：每个 Session 的全部历史（prompt、LLM 流事件、工具调用、token 用量）持久化在 event 表，可按序重放（见 [../6_modules/](../6_modules/README.md) 事件溯源篇）。

### 2.1 bus.log：游标跟随

`core/src/bus.ts:156-176` 的 `log()` 接口注释即规格：

```
readonly log: (input: {
  readonly aggregateID: string
  readonly after?: number          // 排他游标（aggregate seq）
  readonly follow?: boolean        // false: 重放到末尾即完成；true: 重放转直播
}) => Stream.Stream<LogItem>       // Event.Payload | "log.synced" 水位标记
```

`follow: true` 先重放历史再无缝切到 live（`Stream.paginate(sequence, …)`，bus.ts:837-838 分页游标推进，789 行注释：fork 聚合的原始 tail seq 保证游标跨空洞前进）。这就是"复盘"能力：崩溃后从 `after=崩溃前 seq` 重读，精确续传。

### 2.2 对外暴露

- HTTP：`protocol/src/groups/session.ts:726` `GET /api/experimental/session/:id/log?after&follow`（SSE 流，实验性）；
- SSE 总线：`server/src/event-feed.ts`——订阅者队列容量 `SubscriberCapacity = 4096`（10 行），溢出 `SubscriberOverflowError` 直接断流（防慢消费者拖垮发布方）。

### 2.3 花多少钱：session-stats

观测"provider 调用成本"不走日志：`schema/src/session-stats.ts` 定义 `ModelUsage { model, steps, tokens: TokenUsage.Info, cost: Money.USD }` 与 `ToolUsage { calls, succeeded, failed, durationP50 }`——durable 事件折叠出的统计读模型。`TokenUsage.Info`（token-usage.ts:5-12）细分 input/output/reasoning/cache.read/cache.write。

### 2.4 一个复盘走查示例（机制串联）

设想用户报障："昨天那个会话做到一半卡死了"。可用的观测材料与取用顺序：

```
1. run ID 定位    opencode-{channel}.log 里 grep run=<8位>，锁定当天该进程全部日志行
2. 事件重放       GET /api/experimental/session/{id}/log?after=0 → 按序重放全部 durable
                  事件（prompt / llm 流 / 工具调用 / 报错），卡死前最后一条事件即现场
3. (若开了) OTel  按 opencode.run=… 过滤 spans，看 SessionModelTransport.send 时长分布，
                  区分"模型慢"与"本地死锁"
4. (若 mini)      log/direct/{timestamp}-{pid}.jsonl：recv.event 最后一行 vs ui.patch
                  最后一行——服务端停发还是 UI 停绘，一比对即知
5. (若桌面)       marks 随 "app starting" 一行打出，启动段排除/确认
```

这个走查说明本卷的结构性观点：**opencode 的可观测性重心在事件表，不在文本日志**——日志答"进程当时在说什么"，事件表答"会话到底经历了什么"，二者用 run ID / sessionID 各自串联。

---

> OTel/OTLP 概念归 tech-stack；本节讲仓库里的开关、导出内容、埋点面。

### 3.1 开关与资源标识

- **默认关**：`otlp.ts:64` `if (!options?.endpoint) return []`、`:75` `if (!options?.endpoint) return Layer.empty`。endpoint 来自 `OTEL_EXPORTER_OTLP_ENDPOINT`（observability.ts:19），headers 来自 `OTEL_EXPORTER_OTLP_HEADERS`（`k1=v1,k2=v2` 逗号分隔，otlp.ts:16-27 解析）。**没有采样率配置**——endpoint 配了就是全量。
- 资源属性（otlp.ts:45-61）：`service.name="opencode"`、`service.version`、`deployment.environment.name=channel`、`opencode.client`、`opencode.run`/`service.instance.id`（都= runID）。`OTEL_RESOURCE_ATTRIBUTES` 被解析合并（30-43 行，任何条目非法则整体丢弃）；内置键不可被 env 覆盖（有测试：`core/test/effect/observability.test.ts:37` "keeps built-in attributes when env values conflict"）。

### 3.2 双通道

| 通道 | 实现 | 端点 |
|---|---|---|
| logs | `OtlpLogger.make`（effect/observability） | `{endpoint}/v1/logs` |
| traces | NodeSdk + `OTLPTraceExporter` + `BatchSpanProcessor` | `{endpoint}/v1/traces` |

traces 的 OTel 依赖全部**动态 import**（otlp.ts:76-86，五个包并发加载）——不用 OTel 的用户不付启动成本。88-91 行有个关键细节：Effect NodeSdk 不注册全局 context manager，但 **AI SDK 用全局的来给 span 做父子**，所以手动 `new AsyncLocalStorageContextManager()` + `setGlobalContextManager`。

### 3.3 埋点面有多大

埋点不靠手写 `tracer`，靠 Effect 内建：**`Effect.fn("Name")` 与 `Effect.withSpan("Name")` 自动成为 span**。全仓（非测试）`Effect.fn("` 共 **823 处**（grep 实测）——几乎每个 service 方法都自动可观测。抽样（行号实测）：

```
core/src/snapshot.ts:216                Snapshot.boot
core/src/session/model-transport.ts:219  SessionModelTransport.connect
core/src/session/model-transport.ts:389  SessionModelTransport.send     ← LLM 流通道
core/src/plugin.ts:75                    Plugin.load（attribute: plugin.id）
core/src/models-dev.ts:401               ModelsDev.populate
core/src/vcs.ts:167                      Vcs.refreshBranch（attribute: file）
core/src/filesystem/location-watcher.ts:78 LocationWatcher.reconcilе
cli/src/…acp…                            cli.acp.turn.ask / .background
```

即：OTel 打开后，一次"用户发 prompt"的 span 链大致是 `SessionModelRequest.prepare → SessionModelTransport.connect/send → LLM.*`，在 Honeycomb 里按 `opencode.run` 过滤即可看单次会话全链路（这正是 tech-stack honeycomb 条目"宽事件"的用武之地）。

### 3.4 端到端验证

`util/test/observability-otlp.test.ts:5`：起本地 Bun HTTP 假端点，断言 span 真的发到了 `/v1/traces`、自定义 header 透传、body 含 `service.name=opencode` 资源与 span 名——不是 mock 单测，是**协议级冒烟**。

### 3.5 一次启用走查（用法）

部署者视角，把 opencode 接进 Honeycomb 的最小步骤（机制全部来自 §3.1-3.3 实码）：

```bash
# 1. 端点 + 认证（Honeycomb 经典 header 是 x-honeycomb-team）
export OTEL_EXPORTER_OTLP_ENDPOINT="https://otlp.honeycomb.io"
export OTEL_EXPORTER_OTLP_HEADERS="x-honeycomb-team=<API_KEY>"

# 2. (可选) 给这批实例打标签；非法条目会被整体丢弃，小心写
export OTEL_RESOURCE_ATTRIBUTES="service.namespace=myteam,deploy.env=prod"

# 3. 本地日志同步加明细（可选）
export OPENCODE_LOG_LEVEL=DEBUG

opencode serve
```

生效后可查（Honeycomb 宽事件模型下尤其顺手）：

- **哪次 run**：`opencode.run` = runID，一个进程实例一个值——按它 group by 即"单进程会话"；
- **一次 LLM 调用多慢**：`SessionModelTransport.send` span 时长，attribute 里有 connect/send 分离；
- **哪个插件拖慢启动**：`Plugin.load` span 带 `plugin.id` attribute（plugin.ts:75）；
- **日志侧**：`/v1/logs` 同一个 `opencode.run` 资源属性——trace 与 log 在同一后端按 run join。

注意边界：没有 metrics 通道（OTLP `/v1/metrics` 未接）；采样策略不可配（要么全量要么不开），重负载场景慎用 DEBUG 级别 + OTel logs 双开。

---

`server/src/request-tracing.ts` 全文 4 行：

```ts
export const withoutParentSpan = <A, E>(effect: Effect.Effect<A, E, Scope.Scope>) =>
  effect.pipe(Effect.updateContext((context) => Context.omit(Tracer.ParentSpan)(context)))
```

语义（`process.ts:77-78` 装配处注释）："Request fibers may continue inbound trace context, but must not inherit the server startup parent."——从 context 里**删掉 ParentSpan**，于是：

- HTTP 请求 span 不会错误地挂在"服务器启动"这个长 span 下（否则所有请求都是同一个巨 span 的子代，无法区分）；
- 但 `HttpMiddleware.tracer` 会解析入站 **`traceparent` header** 续接上游 trace（W3C 传播）——前端/App 发起带 traceparent 的请求，服务端 span 接上同一 traceId。

`server/test/request-tracing.test.ts:6` 一测双断言（47 行）：无 header 的两次请求 traceId 互不相同且无 parent（43-45 行）；带 `traceparent: 00-1111…-2222…-01` 的请求 traceId 继承 `1111…`、parentSpanId 继承 `2222…`（46-47 行）。

**关联 ID**：跨进程文本日志靠 run ID（§1.1）；跨服务 trace 靠 traceparent；二者独立。

---

## 5. 启动性能测量（marks）

### 5.1 计时点

`desktop/src/main/lifecycle/marks.ts` 全文即一个可变对象，`entry` 在模块加载时立刻打点（先于任何 logger 存在）：

```ts
export const marks: { entry: number } & Partial<Record<
  "ready"|"window"|"served"|"bundle"|"onboarding"|"logging"|"crash"|"storage"|"init"|"layers", number>>
  = { entry: Date.now() }
```

落点（行号实测）：`index.ts:22` ready（Electron whenReady）、`:25` window（早期窗口创建后）、`:31` served（renderer 静态资源服务完一波后）；`desktop.ts:15` bundle、`:20` layers；`native/logging.ts:42` logging、`storage/index.ts:52` storage；`lifecycle/desktop-initialization.ts:32` init；`lifecycle/index.ts:194` onboarding；`native/logging.ts:58` crash（crashpad）。

`native/logging.ts:44-48` 把全量 marks 随 `"app starting"` 日志一行打出——**单次启动也能从日志直接读分段**，不依赖基准工具。

### 5.2 marks 的落点顺序与含义

时间轴上各 mark 的语义（按启动顺序）：

```
entry ──► ready ──► window ──► served ──► bundle ──► onboarding ──► logging
  │         │          │          │          │            │            │
模块加载   Electron   早期窗口   renderer   main bundle  引导决策完成  日志层就绪
(先于一切)  whenReady  创建完成   静态资源    求值完成                  "app starting"
                                 服务完一波                            (全量 marks 随行)
                                                                      │
                                              crash ◄─(crashpad ~60ms，被刻意排在窗口后) │
                                              storage ──► init ──► layers ──► 渲染进程
                                                                                        first paint → shell → idle
```

两个值得记的设计决策（均有源码注释佐证）：

1. **窗口先于 bundle**（`index.ts:23-24` 注释）：模块图求值与 Chromium 初始化抢同一线程，先 import 会拖慢 ready 本身，所以 `import("./desktop")` 排在窗口之后；
2. **crashpad 排在窗口之后**（`native/logging.ts:54-55` 注释）：启动 crashpad 处理进程要 ~60ms 主线程时间，首窗口和它的 IPC 端口优先。

### 5.3 基准测试可断言

`desktop/scripts/bench-startup.ts`（约 570 行）消费这些 marks：

- 隔离 home 目录、剥离开发者 `OPENCODE_*`/`OTEL_*` 环境（文件头 18-20 行注释：绝不碰开发者真实服务与遥测配置）；
- `--runs 5 --warmup 1 --compare B`：交替启动两个构建对冲机器漂移；`--fresh` 测装机后首启、`--service cold|warm` 分冷热服务；
- 里程碑来源四路合流：main 日志（marks）、renderer performance timeline + DOM readiness（CDP 轮询）、Node bootstrap 计时（事后 --inspect 读）、**屏幕像素采样**（`screenPainted`/`screenSettled`——文件头 26-28 行解释：被遮挡窗口 Chromium 停止绘制，splash/淡入会假装"已绘制"，所以直接看屏幕）；
- 输出按 `phaseOrder`（bench-startup.ts:225-245，18 个阶段：electron native init → node bootstrap → entry → ready → window → served → bundle → onboarding → logging → storage → layers → renderer first paint → shell → idle）取**中位数**（247-258 行）。

相关：TUI 侧 `OPENCODE_SHOW_TTFD=1` 在首绘后打印 `startup Nms`（`mini/runtime.ts:741-749`）——终端版的"启动慢在哪"一数字速答。桌面 e2e 性能基准另见 `app/e2e/performance/benchmark.ts`（Playwright trace scope）。

---

## 6. TUI trace 磁道

`opencode mini`（TUI 直连模式）的离线复盘机制。宿主接口 `tui/src/mini/types.ts:155-159` 只有一个 `diagnostics.trace.write(type, data?)`；实现全在 `cli/src/mini-host.ts:46-86`：

- **开关**：`OPENCODE_DIRECT_TRACE` 环境变量（50 行），未设则整个磁道为 no-op；
- **文件**：`{logPath}/direct/{ISO时间戳}-{pid}.jsonl`（55 行），同时写 `direct/latest.json` 指针（59-66 行，含 argv/cwd/path——报障时"发我 latest.json 指的那个文件"即可）；
- **格式**：每行 `{time, pid, type, data}`，BigInt 序列化为字符串（56-57 行）；
- 开写即落一条 `trace.start`（argv/cwd/path，80-84 行）。

磁道事件（行号实测，`mini/` 下）：

```
send.prompt            stream-v2.transport.ts:1738   用户发 prompt（含 delivery 模式）
send.command           stream-v2.transport.ts:1753   斜杠命令
send.shell             stream-v2.transport.ts:1581   shell 类消息
send.permission.reply  runtime.ts:272                权限批复
send.background / send.pending.*  runtime.ts:391/396 后台化 / 挂起项操作
subagent.interrupt / subagent.select  runtime.ts:404/409
session.new            runtime.ts:970
recv.event             stream-v2.transport.ts:1034   收到服务端事件（全量！）
recv.reconnect / recv.reresolve      :1530/:1540    断线重连 / 重新解析失败
ui.patch / ui.commit   runtime.queue.ts:59, stream.ts:133/140/149  终端渲染补丁
```

价值：`recv.event` 记录服务端事件流、`ui.patch` 记录 UI 状态演化——**"用户看到什么"与"服务端发生什么"在同一文件按时间对齐**，离线可逐行复盘终端会话（对照 §2 的服务端 event 表，两端互证）。

---

## 7. 遥测与隐私边界

grep `telemetry/analytics/posthog/sentry` 全仓（非测试）实测结论：**后端（core/cli/server/tui）零使用数据上报**。

- 无 PostHog/分析 SDK（唯一 posthog 命中是 MCP 接入它的 codemode 配置，`core/src/plugin/mcp-codemode-defaults.ts:15`——那是用户自己连 PostHog MCP，不是我们上报）；
- `identity` 包确实只有 logo 资产（`packages/identity/` 全是 svg/png），无遥测 ID——与 [18_credentials.md] 结论一致；
- `schema/src/config.ts:53` `username` 字段描述 "used for telemetry identity" 是**给下游/企业部署的钩子**，core 运行时无消费者（grep `telemetry` 在 core/src 零命中）；
- **唯一例外：桌面 renderer 的 Sentry**——`desktop/src/renderer/startup/sentry.ts:8` `if (!import.meta.env.VITE_SENTRY_DSN) return`，DSN 是构建期注入；prod channel 下还**关闭** GlobalHandlers/BrowserApiErrors（22-29 行，只收显式上报不自动抓全局错误）、去 Breadcrumbs；
- `packages/stats/` 是**公开网站**（模型目录/用量统计，stats.opencode.ai），读 `models.opencode.ai/catalog.json` 匿名聚合，非客户端遥测。

结论：默认安装下没有任何"打回家"的信道；OTel 与 Sentry 都需要部署者显式配置。

---

## 8. 失效场景（观测自身的开销与故障）

| 场景 | 行为 | 依据 |
|---|---|---|
| 日志盘满 | 压缩兜底：>50MB 砍到 ~25MB；追加失败时 Effect Logger 容错不炸主流程 | logging.ts:9-10；trim 1 小时一次，实测 1GB 修剪 <1s（:12 注释） |
| 多进程同时 trim | mkdir 原子锁，输者本轮让路；死进程锁 5 分钟后可破 | logging.ts:93-97, 133-142 |
| trim 与追加竞态 | 接受毫秒级尾部丢失（不引入跨进程写锁） | logging.ts:83-85 注释 |
| OTel 端点不可达/初始化失败 | `Layer.catchCause(() => local)` 整层回退纯本地日志；子进程级测试锁定 | observability.ts:50；test :51 "falls back to local logging when OTLP initialization fails" |
| OTel 运行中发不出去 | BatchSpanProcessor 批处理缓冲（OTel SDK 行为，仓库不额外处理） | otlp.ts:95 |
| SSE 慢消费者 | 队列 4096 上限，溢出 SubscriberOverflowError 断流不拖垮 bus | event-feed.ts:10-14 |
| 性能开销 | OTel 五个依赖全动态 import（不配 endpoint 零成本）；fileLogger 有 batchWindow（logging.ts:65 注释：置 0 会导致空闲 CPU 飙升，禁改） | otlp.ts:76-86, logging.ts:65 |
| workerd 全局取随机 | runID 懒生成 + Layer.suspend | shared.ts:1-2, observability.ts:52-53 |

---

## 9. 测试覆盖

| 测试文件 | 覆盖 |
|---|---|
| `core/test/effect/observability.test.ts`（7 test） | resource 属性解析/容错/内置优先（:18/:30/:37）；OTLP 失败回退本地（:51）；并发 run 每行带 run ID（:87）；嵌套对象拍平（:115） |
| `util/test/observability-otlp.test.ts` | 协议级冒烟：span 真发 `/v1/traces`、header 透传、资源属性（:5） |
| `server/test/request-tracing.test.ts` | 无 parent 起始 + traceparent 续接，一测双断言（:6, 43-47） |
| `core/test/event-logger.test.ts` | 5 类 updated 事件转日志（:18） |
| `core/test/logging.test.ts` | local/non-local 日志文件名（:7/:11） |
| `desktop/scripts/bench-startup.ts` | 非测试：marks 的唯一自动化消费者，冷启动可断言基准 |

注：`Logging.trim` 本身无直接单测（grep 全仓无 `Logging.trim` 测试引用），其正确性靠注释里的时序论证 + 生产运行验证——是观测件里测试最薄的一环。

### 演进注记（2026-10-22 快照）

- `session.log` HTTP 端点带 `experimental` 前缀，durable 事件对外 API 仍在实验期；
- OTel 无采样配置（全量或全无）——大规模部署若需采样，得改 otlp.ts；
- bus `listen` 已标 `@deprecated`（bus.ts:172），观测消费方向收敛到 `subscribe()`/`log()` 流式接口。

---

## 相关篇

- [../3_ops/1_log-locations.md](../3_ops/1_log-locations.md) —— 日志**在哪**（本篇讲**怎么写/怎么压**）
- [../2_tech-stack/2.4_infrastructure/7_opentelemetry.md](../2_tech-stack/2.4_infrastructure/7_opentelemetry.md) / [8_honeycomb.md](../2_tech-stack/2.4_infrastructure/8_honeycomb.md) —— OTel 与观测后端**是什么**（本篇讲仓库**怎么用**）
- [../6_modules/](../6_modules/README.md) —— 事件溯源 / bus / Session 执行（§2 的深层机制）
- 本卷 [1_durability]（事件持久化）、[9_performance]（启动预算）、[10_lifecycle]（关闭与恢复）
