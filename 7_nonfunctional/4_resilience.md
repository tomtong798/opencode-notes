# 断线与降级（Resilience）

> 一句话：**opencode 对"连接会断、依赖会挂"的回答是把韧性拆成四把伞——SSE 连接层的握手/心跳/watchdog/退避重连状态机（客户端自治）、断线后的"失效重取"策略（全键失效 + refetch，而非事件补发）、服务端慢消费者的"只断溢出者"（dropping 队列 4096）、依赖侧的分类降级（LLM 错误重试矩阵 / MCP 会话过期一次性重连 / models.dev 快照下限）——每一层都假设下一层会失败，没有一层假设网络是可靠的。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-22，sed 抽查 ≥15 处）。
> 权威出处：`packages/client/src/solid/connection.ts`（重连状态机全文 277 行）+ `packages/server/src/event-feed.ts`（慢消费者处理全文 89 行）。
> [返回 README](README.md) | 关联 [../5_timeline/4_renderer-boot.md](../5_timeline/4_renderer-boot.md) §3 / [../5_timeline/6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md) / [5_resource-limits.md](5_resource-limits.md) §2b / [3_idempotency.md](3_idempotency.md) / [../6_modules/14_mcp.md](../6_modules/14_mcp.md) / [../6_modules/3_model-protocol.md](../6_modules/3_model-protocol.md) / [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5
> 本篇是**韧性视角的收口**：renderer 重连状态机（4_renderer-boot §3）、事件背压（5_resource-limits §2b）、重试矩阵（3_idempotency）已各有专篇，这里只补"它们如何拼成一张断线自愈的总图"，并引入此前未展开的实码：页面生命周期韧性、三选一状态策略对照、依赖降级路径、竞争者池。

---

## 0. 恐惧清单（这个产品怕什么）

```
opencode = 长连接 + 多依赖的本地客户端
├─ 恐惧 1：WiFi 切换 / 网络抖动留下"半开连接"——TCP 还活着，字节永远不来
│    └─ 防线：15s 心跳 + 45s watchdog（字节级活性，非连接级）（§1）
├─ 恐惧 2：服务升级 / sidecar 重启瞬断——事件缺口期间 UI 状态变陈旧
│    └─ 防线：失效重取——断开即全键失效，重连后 refetch 对账（§2）
├─ 恐惧 3：后台标签页被浏览器冻结——timer 全停、socket 死了无人报
│    └─ 防线：pagehide/pageshow + 回前台空闲 20s 强断 resync（§3）
├─ 恐惧 4：一个慢设备（旧手机/卡死的 tab）拖死整个事件广播
│    └─ 防线：dropping 队列 4096，满则只断溢出的那一个订阅者（§4）
├─ 恐惧 5：LLM provider 限流 / MCP server 掉线 / models.dev 不可达
│    └─ 防线：三套互不相同的降级——重试矩阵 / 会话过期重连 / 快照下限（§5）
└─ 恐惧 6：两个客户端同时冷启，双拉后端服务
     └─ 防线：contender 竞争者池——至多 2 个竞争者，谁先注册谁赢（§6）
```

贯穿全篇的一个设计立场：**客户端从不请求事件补发**。SSE 流是易失的（volatile），断线缺口用"重取权威状态"弥补，而不是"回放 missed events"——这是 §2 三选一里最重的一个架构决定，甚至有专门的回归测试钉住它（§8）。

另一条暗线：**所有恢复路径的终点都是"重取权威状态"，而不是"修复局部状态"**。半开连接不试图探测修复（直接断掉重连）；慢消费者不试图限速追赶（断掉重连后 refetch）；页面冻结不试图区分 socket 死活（超阈值就当死）。放弃"温和修复"换来的是每条恢复路径都极短、极可测——状态机的每一个分支都能在单元测试里走通（§8）。

---

## 1. 连接生命周期全景

[4_renderer-boot §3](../5_timeline/4_renderer-boot.md) 已画重连状态机的运行流；本节补"一生"的时间轴与容差设计。

### 1.1 一生时间轴（connection.ts + handlers/event.ts 实测）

```
t=0      connect() 发起 GET /event
          └─ 2s 连接超时兜底（connection.ts:32 connectTimeout）
t≈0+     服务端先发一条 server.connected 帧（event.ts:13-19）
t≈0+     客户端校验首个事件必须是 server.connected
          （connection.ts:106-107 否则报 "did not start with" 直接断）
          → 状态 connected，attempt 归零，touch() 启动 watchdog
稳态      每 15s 一条 ": heartbeat" 注释帧（event.ts:21 Stream.tick）
          每个到达的字节（含心跳）都 touch() → watchdog 重置
          （connection.ts:84-91 "Any received bytes, including keepalive
           comments, push the stall deadline out"）
异常      45s 无任何字节 → watchdog abort("Event stream stalled")
          （connection.ts:35 defaultIdleTimeout = 45_000）
断        状态 reconnecting，attempt+1，record("disconnected") 进历史环
          （connectionHistoryLimit 50，connection.ts:34）
退避      wait(1s) 后重连（connection.ts:33 reconnectDelay）
          ├─ 存活 ≥1s 的连接把 attempt 归零（:158 "长连成功可重置退避"）
          └─ resync 触发的强断跳过退避立即重连（:176-180 forced 注释：
             "A deliberate resync already knows the old socket is gone"）
重连      回到 t=0——新连接、新握手、新 watchdog
```

注意退避是**固定 1s 不是指数**：attempt 只用于展示（UI 上"第 N 次重连"），唯一会拉长的是 contender 场景下的服务再拉起时间。本地 sidecar 场景网络往返 <1ms，指数退避只会让"服务重启后多等几秒"——1s 是为本地进程调的参数。

### 1.2 握手协议：为什么首事件必须是 server.connected

`handlers/event.ts:11-23`（服务端）与 `connection.ts:97-115`（客户端）共同构成一个最小握手协议。服务端把订阅者队列与一条预制的 `server.connected` 帧拼接（`event.ts:18-19` `Stream.make(frame(connected)).pipe(Stream.concat(live))`）——**先占位再订阅**，保证订阅建立与首帧之间没有事件缝隙（若先订阅再发首帧，缝隙期事件会先于首帧到达，客户端的"首事件校验"就无意义了）。客户端 `connection.ts:106-107` 硬校验：

```typescript
if (first.value.type !== "server.connected")
  return { error: new Error("Event stream did not start with server.connected"), connectedAt }
```

这条校验看似多余（自家服务端总会发），实际防的是**中间层劫持**：代理、auth 网关、错误页都可能在 SSE 流前面塞别的东西。首帧不对就当坏连接立即断——fail-closed，与 [6_security.md](6_security.md) 的边界纪律同源。握手成功后 `connectedAt = Date.now()`（`:111`），它是后续两个判定的基准：watchdog 从此刻起算（`touch()` 只在 `connectedAt !== undefined` 时挂 timer，`:89`——连接期不判静默，2s 连接超时管这一段）；attempt 归零判定用它（`:158` 存活 ≥1s 才归零，握手即断的连接不算成功）。

### 1.3 为什么 watchdog 是 3× 心跳宽

三个数字的关系（全部实测）：

| 参数 | 值 | 出处 | 语义 |
|---|---|---|---|
| 心跳间隔 | 15s | `event.ts:21` `Stream.tick("15 seconds")` | 服务端保活周期 |
| watchdog | 45s = 3×心跳 | `connection.ts:35` | 连接级无字节判死线 |
| 前台空闲阈 | 20s ≈ 1×心跳+5s | `connection.ts:38` | 回前台时判"半开"的判死线 |

45s = 3× 的容差设计：watchdog 要防的是**连接级死亡**（半开 TCP、NAT 表项过期），不是**调度抖动**。心跳是 `Stream.tick` 定时器，事件多时与业务帧交错、事件运行时 GC 停顿、容器 CPU 限流都可能让一拍心跳迟到几百毫秒甚至丢一拍。3× 意味着**连续错过两拍心跳 + 第三拍迟到**才判死——把"误杀健康连接"（代价：无谓重连 + 全量 refetch）压到接近零，同时 45s 仍是人类可容忍的"UI 显示断线"上限之内。反过来，`foregroundIdleThreshold`（:36-37 注释原文 "Longer than one server keepalive interval"）只需要比一拍心跳宽就能在**回前台瞬间**判定"设备睡眠期间 socket 已死"——因为此时要的是快，且判错的代价只是多一次 resync（幂等）。

另一个细节：`touch()` 用的是**字节级活性**（`onActivity` 回调，连 SSE 注释帧都算），不是"收到完整事件"——半开连接连心跳都送不来，这正是 watchdog 的目标。

### 1.3 resync：不是断线的断

`connection.ts:212-217` 的 `resync(reason)` 是主动强断：`forced = true` + 立即 abort 当前请求。它存在的理由是**有些死亡浏览器不会报告**（§3），与其等 45s watchdog，不如在"已知大概率已死"的时刻主动断。注释原文（:211）："Drop the live request so the reconnect loop replaces it now instead of waiting for the idle watchdog"。且只在 `connected` 状态生效（:213）——未连接时没有可断的东西。

### 1.4 重连风暴的削峰三件套

服务重启时所有客户端同时断线、同时重连、同时 refetch——三道削峰（全部实测）：

1. **attempt 归零而非指数退避**（`:158`）：本地 sidecar 重启通常 <1s，固定 1s 重连让风暴窗口只有一轮；若指数退避，各客户端因 attempt 不同错峰——但代价是部分客户端多等几秒才恢复，对本地进程不划算。
2. **`refetchType: "none"`**（`sync.tsx:177`）：失效只标脏，重连握手（而非失效瞬间）才触发重取——反复断连期间失效被合并成最后一次。
3. **目录两两排队 + held 优先**（`sync.tsx:182-186` + `connection.ts:26-28`）：重连后不是全部目录并发 resync，而是刷新队列一次吐两个、当前挂载的目录插队——注释原文（`sync.tsx:182-183`）："Syncing every active directory here as well sent the whole catalog fan-out for all of them at once"。

---

## 2. 断线期间的状态策略三选一（核心交付）

断线必然留下事件缺口。业界三种弥合策略，opencode 在不同客户端做了不同选择：

| 策略 | 机制 | 谁在用 | 适用场景 | 代价 |
|---|---|---|---|---|
| **A. 事件补发**（durable log replay） | 客户端带 last-event-id / seq，服务端从 durable log 回放缺口 | `sessions.log`（实验性 `session.log` 端点，`protocol/src/groups/session.ts:726-747`：`after` seq + `follow=true` 续 live） | 需要**无损**事件流的消费者：执行恢复、审计、外挂投影 | 服务端必须为每条事件编 seq 并持久化（durable 事件带 `aggregateID+seq`，`connection.ts:127-132` 调试日志可见）；重放放大读流量；live-only delta（文本增量）不保证重放 |
| **B. 失效重取**（invalidate + refetch） | 断开即全键失效，重连握手后 refetch 权威快照 | **app（桌面/Web）**——`server-sync/connection.ts:9-12`：状态 ≠ connected 的瞬间 `invalidate()`；重连后 `sync.tsx:180-187` bootstrap refetch + 按目录排队 resync | 状态可从服务端 API 完整重取、且服务端本来就是权威 | 重连风暴（N 个 query 同时 refetch）；缺口期间 UI 显示的是**陈旧但一致**的快照 |
| **C. 投影对账**（reconcile prefix diff） | 客户端持有已渲染前缀，重取全量后按前缀比对、只补差量 | **CLI 非交互模式**——`noninteractive.ts:568-619`：`projectedMessages()` 重取消息，逐条与 `renderedText`/`renderedTools` 前缀比对（`:580` `item.text.startsWith(rendered)` 则 `slice` 补尾部） | 单向输出、需要**精确续显**（stdout 不能重复不能跳行）的场景 | 对账逻辑侵入渲染层；只适合线性 append 的投影 |

**app 为什么选 B**：`sync.tsx:174-179` 的失效实现刻意保守——`invalidateQueries({ predicate: scope, refetchType: "none" })`，`refetchType:"none"` 意味着**先只标脏不立刻重取**（断开期间反复失效不产生请求风暴），真正的重取发生在重连握手（`server.connected`）之后：bootstrap refetch + `reconnectOrder`（`connection.ts:26-28`，挂载中的目录优先）按队列两两同步（`sync.tsx:182-183` 注释："The refresh queue re-syncs two directories at a time, held ones first"——防止全目录 fan-out 一次性打爆服务端）。加上 `stop()` 里的注释（`connection.ts:207`）："consumers must treat their data as stale until start() reconnects"——**陈旧是显式契约，不是事故**。

**三者的共同底座**：服务端是权威，客户端投影是缓存。A 把权威做成可寻址的事件流（最重），B 把权威做成可重取的快照（中），C 把权威做成可 diff 的前缀（最轻但只适用于线性投影）。TUI 侧的等价物是 hydrate 六路并发重放（连接/重连/resize 共用，见 [6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md)），语义上同属 B/C 混合。

重试幂等性（重取会不会造成重复副作用）由 [3_idempotency.md](3_idempotency.md) 的铸 id + 服务端 reconcile 承担，两篇互为表里：本篇管"连接层怎么回来"，那篇管"回来之后重放的请求怎么去重"。

三策略的对照读法：**代价随精确性递增**。C（前缀对账）最精确也最贵——对账逻辑长进渲染层，只服务 stdout 这种"错一个字节都不行"的单向流；A（事件补发）把精确性做成服务端能力（durable seq），客户端便宜但服务端要维护事件日志与游标协议；B（失效重取）最便宜——服务端零新增（API 本来就在），客户端只需信任"重取能收敛"。app 的投影（会话列表、消息、MCP 状态）天然可从 API 重取，所以 B 够用；CLI 的 stdout 无法重取（已经印出去了），所以必须 C；需要字节级无损的消费者（执行恢复、外挂投影）才值得 A。

---

## 3. 页面生命周期韧性

`connection.ts:219-240`（`pageLifecycle: true` 时挂载，app 默认开启，`4_renderer-boot.md` §3 的 `client.tsx:92-105` 装配点传的就是它）：

```typescript
// connection.ts:219-233（节选）
const pagehide = () => stop()
const pageshow = () => void start()
// Locking a phone or switching apps hides the document without a pagehide; the socket
// usually dies while the page is suspended, and the browser may never report that on the hung read.
const visibility = () => {
  if (document.visibilityState !== "visible") return
  if (Date.now() - lastActivity < foregroundIdleThreshold) return
  resync("Page returned to the foreground after the event stream went quiet")
}
const online = () => resync("Network connection restored")
```

四个事件各防一种死法：

| 事件 | 行为 | 防什么 |
|---|---|---|
| `pagehide` | `stop()`：generation+1、abort 流、状态回 connecting | 移动端杀后台前释放连接，省电省服务端订阅 |
| `pageshow` | `start()`：新 generation 重连 | bfcache 恢复的页面拿回一条新鲜连接 |
| `visibilitychange` → visible | 空闲 >20s 则 `resync()` 强断 | **手机锁屏/切 app 不触发 pagehide**（:222-223 注释），socket 在冻结期死了，浏览器对挂起的 read **永远不报错**——只能回前台时主动判 |
| `online` | `resync()` 强断 | WiFi 切换：`online` 事件来时旧 socket 大概率半开，与其等 watchdog 不如立刻换 |

**"suspended fires as soon as resumes" 的坑**（:84-85 注释原文）："A timer whose deadline passed while the page was suspended fires as soon as the page resumes."——浏览器冻结期间 `setTimeout` 不走，但**解冻瞬间到期 timer 会立即触发**。若 watchdog 是 45s 而页面冻结了 10 分钟，回前台的一瞬间 watchdog 立刻 abort——这本身无害（本来就要重连），但 `visibility` 处理器与它**赛跑**：`visibilitychange` 先查 `Date.now() - lastActivity < 20s` 才 resync，避免与已触发的 watchdog 重复强断。20s 阈值同时把"冻结 <20s（大概率 socket 还活着）"排除在外——短冻结不必浪费一次全量 refetch。

这个坑的另一半在 `touch()` 的实现里：`lastActivity` 在**冻结前最后一次收字节**时写入，而 `Date.now()` 在解冻后读取——两者横跨冻结期，差值天然包含冻结时长。于是 20s 阈值的真实语义是"距最后一次字节活动（含冻结期）不足 20s 就不动"——不是"回前台后观察 20s"。判定是即时的、零等待的，这在用户体验上至关重要：回前台要么 0ms 什么都不做（连接还活着），要么 0ms 强断重连（连接死了），**永远不出现"回前台后干等 20s 才发现连接死了"**。

`stop()` 的状态语义（:207 注释）：停止后**没有人在听**，所以直接把状态置回 `connecting` 而不是 `disconnected`——`disconnected` 会触发 app 层失效（§2 的 effect 对一切非 connected 状态生效），而 pagehide 场景下这个失效是必然要发生的，等 pageshow 后第一次 `connecting → connected` 握手再对账即可。

代际交接防双流：`stop()` 里 `generation += 1`（`:205`），正在飞的 `runStream` 循环的每个 await 点都检查 `generation !== active` 即静默退出（`:157`）——不会与 pageshow 起的新循环形成双流；`start()` 里 `await previous` 再接棒（`:191-193`），两代循环串行交接。这是整个状态机里唯一的并发控制：没有锁，只有一个单调递增的代数号。

---

## 4. 服务端广播的慢消费者处理

完整闭环在 [5_resource-limits.md](5_resource-limits.md) §2b（队列容量、客户端二次防御都有）；本篇只补"被断者的恢复路径"这半环，凑成韧性视角的完整链：

```
server 广播（event-feed.ts:64-68 实测）
└─ for (const subscriber of subscribers)
     ├─ Queue.offerUnsafe 成功 → continue（快者无感）
     └─ 失败（dropping 队列 4096 满）
        ├─ subscribers.delete(subscriber)   ← 只踢这一个
        ├─ Queue.failCauseUnsafe(SubscriberOverflowError)
        └─ SSE handler 写出错帧后结束响应
              ↓
被断的客户端（§1 状态机接管）
└─ iterator done → record("disconnected") → 1s 后重连
   → server.connected 握手 → §2 失效重取补缺口
      （gap 期间的状态用 refetch 对账，不需要事件回放）
```

关键设计仍是"**只断溢出者，不殃及快者**"：`publish` 对每个订阅者独立 offer，慢者的队列满只导致它自己被移出 `subscribers` 集合（`event-feed.ts:66`），发布循环继续。被断者走的是与网络断线**完全相同**的恢复路径——服务端不需要为慢消费者做任何特殊处理，客户端也不需要知道自己为什么被断。这是 §2 选 B（失效重取）的另一个红利：**所有断线原因（网络/溢出/服务重启/页面冻结）汇聚到同一个恢复协议**。

还有一个容易忽略的边界：编码失败。`render(event)`（JSON 序列化）若抛错，`event-feed.ts:41-46` 的 `fail()` 会把**全部**订阅者 fail 掉（`subscribers.clear()` 后逐个 `failCauseUnsafe(EncodingError)`）——与慢消费者的"只断一个"刻意不同。理由是编码失败意味着这条事件**对所有订阅者**都编不出来（不是某个消费者慢），留下任何活着的订阅者都意味着它的事件流有一个永久空洞，还不如全断全重连。全局故障全局处理，局部故障局部处理。

---

## 5. 依赖降级

三依赖、三套降级，共同点：**先分类，再决定**——从不盲目重试。

### 5.1 LLM provider 不可用

两层结构：**分类器**（`ai/src/provider-error.ts:235-305` `classifyProviderFailure`）把混沌错误归一为带类型的 AIError；**重试策略**（`core/src/session/runner/retry.ts`）按类型决定重试与否。

分类的立场（`provider-error.ts:229-234` 注译）：确定性失败需要**肯定性证据**（4xx 状态、quota/auth/policy 信号）才落对应类；识别不了的落 `UnknownProvider`——而它**可重试**，因为"瞬时失败恰恰以分类器预料不到的形状到来"（:98-100 注释原文）。`isRetryable`（`:81-116` 实测）：

```
可重试   RateLimit / ProviderInternal / UnknownProvider
         Transport（未拒收，或已受理但为读操作——:89-94 注释：
           "A WebSocket acknowledgment marks delivery accepted
            before model output may exist"）
         InvalidProviderOutput 且 classification === "incomplete-stream"
不可重试 Authentication / QuotaExceeded / ContentPolicy /
         InvalidRequest（含 context-overflow / payload-too-large）/
         UnsupportedOperation / NoRoute / Timeout
覆盖口   x-should-retry 响应头一票定夺（:82-84）
```

重试调度（`retry.ts:42-56`）：jittered 指数退避 2s 起步、单 gap 封顶 10s、共 10 次（全失败约 84s 等待）；尊重 provider 的 `retry-after` 但 **15 分钟封顶**（`:31-32` 注释原文："Bound provider-requested delays so a hostile or buggy retry-after cannot stall a session for hours"）；超时类单独 **3 次封顶**（`:58-60`："a dead provider hold(ing) a step for most of an hour"）。每次重试发 `SessionEvent.RetryScheduled`（`:104-110`）——**用户可见**：UI 能显示"第 N 次重试、预计 X 秒后"，而不是转圈静默。

**实测结论：没有自动模型降级**。grep 全仓未发现"provider 失败 → 换模型"的路径；`step.ts:185` 的 `rotate-and-retry-full` 是 **WebSocket 通道轮换**（transport 层换连接，`model-transport.ts:318` `rotateAfterMs`），不是模型 fallback。重试耗尽后 `publisher.failAssistant(llmError)`（`step.ts:211`）——错误如实落进会话成为 durable 事实，换模型是**用户**的决策（显式 retry 并换模型，见 [3_idempotency.md](3_idempotency.md) §1.1 客户端 retry 状态）。这是诚实的设计：静默降级会让用户以为在用 Claude 而实际在用别家。

### 5.2 MCP server 掉线

机制详见 [14_mcp.md](../6_modules/14_mcp.md)；韧性视角收口三点（`core/src/mcp/index.ts` 实测）：

1. **会话过期的一次性重连**：`recovering()` 包装器（`:350-366`）对请求捕 `MCP.SessionExpiredError` → stop+start 重连 → **同请求重跑一次**（注释原文 ":349"："reconnects and runs it once more against the replacement. Any other failure passes through"）。只重一次、只针对会话过期——不做无限重试。
2. **后台通知也触发**：`connection.onSessionExpired(() => fork(recover(...)))`（`:414-416`，注释："Background refreshes (list-changed) can observe the expiry too; they do not retry, so reconnect here for them"）。
3. **状态机层不自动重试**：连接失败落 `failed`/`needs_auth` 状态广播，恢复路径是用户从 `/mcps` 手动重连，或 OAuth 凭证切换事件（`Credential.Event.Switched` → `:579-596`）自动 reconnect。**MCP 是用户装的第三方，挂了让用户看见**——与 LLM 的静默重试形成刻意对比。

### 5.3 models.dev 不可达

目录服务的四级降级链（`core/src/models-dev.ts` 实测，机制详见 [3_model-protocol.md](../6_modules/3_model-protocol.md) §2）：

```
populate（:385-401）按序回退
  ① options.file（测试/私有部署指定文件）
  ② KV 缓存（:333-343，上次成功拉取的 catalog 全文 + digest）
  ③ ★bundled snapshot（:281-297，仓库内置 5.7MB 快照，
     注释 :361-363 "bundled snapshot is the boot-time floor"）
  ④ 网络拉取（加锁防并发，:393-399 Semaphore 串行 double-check）
```

网络层的韧性预算很克制：`retryTransient` 仅 **2 次**、200ms 起 jittered 指数（`:318-321`），单请求 10s 超时（`:350`）；成功结果写 KV 时**失败不致命**（`:365-376` 注释："a cache-write failure must never kill catalog population"，且注释承认 payload 已超过部分 KV 后端 2MB 单值上限——"a boot without a cache hit just refetches"）。周期刷新 5min TTL（`:330`）+ digest 比对避免无谓重写。**净效果：断网也能列出全部模型**（快照下限），只是新模型要等网络回来。

---

## 6. 竞争与接管

多客户端（桌面 + TUI + CLI）并发冷启时谁拉起后端服务？答案是**竞争者池**（`client/src/service-contender.ts:61-118`，spawn 语义与启动时序见 [1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5）：

```
ensure() 发现服务缺失
└─ contenderPool（service-contender.ts:61-118）
   ├─ shouldRecruit（:104-108）：failure === undefined
   │    && contenders.size < 2 && 距上次 spawn ≥ spawnDelay
   │    （注释 :106-107 "Keep one candidate plus one lock probe for
   │      pre-lock stalls"——第二个竞争者是探锁探针）
   ├─ 谁先写出注册文件谁赢；输家走 reuse 分支直接采用
   ├─ 干净退出（exitCode 0）= 别人赢了
   │    → spawnDelay ×2 退避（:95-96，封顶 maxSpawnDelay）
   │    → 防两个客户端互相反复 spawn
   ├─ evict(pid)（:77-86）：在位服务被替换 → 清它的竞争者、
   │    清失败记忆、重置 spawn 时钟
   ├─ reap()（:92-99）：全部竞争者死光才上报 failure——
   │    且失败后不再补员（:107 注释："let the survivors finish without
   │    recruiting replacements that could hide the error indefinitely"）
   └─ serviceAnswered()（:70-72）：探活命中 → 退避归零
```

韧性语义有两层。**双启免疫**：至多 2 个 contender + 注册文件单写者，任意时刻全机每版本至多一个真服务。**错误不掩盖**：竞争失败不上报（那是正常的输），启动失败只在"没有任何存活尝试"时上报一次（`reap` 的 `failure ??=` 记住**第一个**失败），且失败后停止招募——宁可让用户看到"服务起不来"，不让无限补员把根因藏进噪声。双客户端抢同服务对上层完全透明：输家 ensure 返回的时间只比赢家多一次注册文件轮询，SSE 层（§1）对"服务是谁拉的"一无所知。

与断线恢复的衔接：在位服务崩溃时，注册文件因探活失败（3 次超时，`1_desktop-boot.md` §1.5）而失效，各客户端下次 ensure 重新进竞争——**接管与冷启是同一个协议**，没有专门的"崩溃恢复路径"。renderer 侧对服务替换的感知同样收敛：`AppReconnectService` RPC 拿新 URL 后走的就是 §1 的重连循环（`1_desktop-boot.md` §2.5 明言"桌面端不特殊处理，复用与 Web/TUI 相同的连接语义"）。

---

## 7. 失效场景矩阵

| # | 失效场景 | 撞上的防线 | 残余风险 |
|---|---|---|---|
| 1 | WiFi 切换留半开连接 | `online` 事件 resync 强断（§3）+ 45s watchdog 兜底（§1） | 切换后 <20s 内的短冻结不触发 resync——watchdog 兜底，最长 45s 陈旧 |
| 2 | 服务升级瞬断 | disconnected → 1s 重连 → server.connected 握手 → 全键失效+refetch（§2） | 重连风暴由 `refetchType:"none"` + 两两目录队列削峰（`sync.tsx:176-187`） |
| 3 | 后台标签页冻结 | visibilitychange 回前台 >20s 空闲 → resync（§3） | 冻结 <20s 赌 socket 还活着——赌错由 watchdog 45s 收尾 |
| 4 | 慢设备拖死广播 | dropping 队列 4096 只断溢出者（§4） | 被断者重连后 refetch——慢者反复被断（设计如此：慢是它自己的问题） |
| 5 | LLM 429 限流 | RateLimit 可重试 + retry-after 尊重但 15min 封顶 + RetryScheduled 用户可见（§5.1） | 10 次重试耗尽 → failAssistant 落 durable 错误，无自动模型降级（用户决策） |
| 6 | LLM 认证/配额/内容政策失败 | 不可重试分类 → 直接 failAssistant（§5.1） | 无——重试只会浪费钱 |
| 7 | MCP server 会话过期 | recovering() 一次性重连重跑（§5.2） | 重连后再失败 → 错误透传 |
| 8 | MCP server 启动失败 | 状态机落 failed 广播，不自动重试（§5.2） | 用户手动 /mcps 重连（第三方进程，静默重试可能掩盖根因） |
| 9 | models.dev 断网 | KV 缓存 → bundled snapshot 下限（§5.3） | 新模型目录缺失直至网络恢复 |
| 10 | 双客户端并发冷启 | contender 池 ≤2、输家退避、赢家注册（§6） | 极端下短瞬双进程，注册文件单写者保证快速收敛 |
| 11 | 断线期间事件缺口 | 失效重取（§2 B）——客户端不请求补发（`solid-connection.test.ts:166` 钉死：重连**不带** last-event-id） | live-only delta 丢失由 refetch 权威快照吸收 |
| 12 | 恶意/buggy retry-after | RETRY_AFTER_MAX 15min 封顶（`retry.ts:31-32`） | 无——会话最长被拖 15min/次 |
| 13 | 事件编码失败（JSON 序列化抛错） | fail() 全体断开重连（`event-feed.ts:41-46`，§4 末） | 全体客户端一次断连风暴——设计认定编码失败是全局故障 |
| 14 | LLM 请求已受理但连接断（WebSocket ack 后断） | Transport + delivery:"accepted" 且读操作 → 可重试（`provider-error.ts:89-95`） | 写操作不重试——防重复副作用（与 3_idempotency §重试矩阵衔接） |

矩阵读法：**1-4 是连接层的自治自愈（用户最多看到一闪而过的"重连中"）；5-9 是依赖层的分类降级（该重试的重试、该认输的认输、该让用户看见的让用户看见）；10 是竞争协调**。没有一层试图把不可靠变成可靠——每层只承诺"在有限时间内回到一致状态"。

---

## 8. 测试覆盖（实测）

```
防线                        测试文件（实测存在）
──────────────────────────────────────────────────────────────
SSE 重连状态机              packages/client/test/solid-connection.test.ts
  ├─ :70  静默超 watchdog 被替换（断言 "Event stream stalled" 历史）
  ├─ :94  心跳注释帧维持静默连接存活（250ms 高频心跳不被误杀）
  ├─ :115 resync 立即换流且仅 connected 态生效（<500ms 断言）
  ├─ :141 流被服务端关闭/出错 → reconnecting 且记录原因（参数化两路）
  └─ :166 ★重连不请求事件回放（断言新流 headers 无 last-event-id）
                              ——把 §2 "不补发"的架构决定钉进契约
失效重取（app）             packages/app/src/runtime/server/server-sync/connection.test.ts
  ├─ :4   held 目录先刷新、其余保持原序（reconnectOrder）
（e2e 侧）                  packages/app/test-browser/server-connection-sync.test.ts
  └─ :6   断开即失效、每次握手后同步（§2 的 effect 契约）
竞争者池                    packages/client/test/service-contender-pool.test.ts
  ├─ :18/:35 首个失败记忆 / evict 清记忆重启时钟
  ├─ :53/:63 未答注册让一拍 / 至多两尝试存活
  └─ :75/:92 干净退出退避到上限 / 服务应答重置退避
LLM 重试分类                packages/ai/test/（provider-error 相关 cassette 回放）
MCP 重连                    packages/core/test/mcp.test.ts（详见 14_mcp §测试）
durable 事件回放            packages/core/test/session-log.test.ts
                              （sessions.log 的 after/follow 语义——策略 A 的主人）
慢消费者（服务端）          event-feed 队列语义 → 5_resource-limits §2b 已收口
models.dev 降级链           packages/core/test/（models-dev 缓存/快照/fetch 回退路径）
e2e 断网                    packages/app/e2e/（service-worker/cache.spec.ts 等，
                              mock-server.ts + sse-transport.ts 提供可操纵的假 SSE：
                              测试可按需 write/close/error 一条 SSE 流、
                              检查每条流的 aborted 状态与请求头）
```

测试基建值得一提：`e2e/utils/mock-server.ts` 与 `sse-transport.ts` 把"一条 SSE 连接"抽象成可编程对象（`fake.streams[n].write/close/error`），使 `solid-connection.test.ts` 能用 60ms 的 watchdog 跑完"45s 静默判死"的等价场景——时间参数可注入（`idleTimeout` option），但**状态机本身跑的是生产代码**，无 mock。

值得单记的是 `solid-connection.test.ts:166` 那条：它断言的不是"重连成功"，而是**重连时不得携带 last-event-id**——把"事件流是易失的、缺口由 refetch 弥补"这个 §2 的核心决定变成了回归测试。谁想给 SSE 加回放，先得改这条测试。

---

## 演进注记

### 附：韧性参数速查表（全部实测）

| 参数 | 值 | 出处 |
|---|---|---|
| 连接超时 | 2s | `connection.ts:32` |
| 重连间隔（固定，非指数） | 1s | `connection.ts:33` |
| watchdog（无字节判死） | 45s = 3× 心跳 | `connection.ts:35` |
| 前台空闲阈值 | 20s ≈ 1× 心跳+5s | `connection.ts:38` |
| 服务端心跳 | 15s | `handlers/event.ts:21` |
| 慢消费者队列 | 4096 | `event-feed.ts:8` |
| 连接历史环 | 50 条 | `connection.ts:34` |
| LLM 退避 | 2s 起、单 gap ≤10s、10 次 | `session/runner/retry.ts:42-48` |
| retry-after 封顶 | 15min | `retry.ts:32` |
| 超时类重试封顶 | 3 次 | `retry.ts:60` |
| models.dev 刷新 TTL | 5min（拉取重试仅 2 次、单请求 10s） | `models-dev.ts:330,318-321,350` |
| contender 池上限 | 2（一候选 + 一探针） | `service-contender.ts:108` |

对照旧卷：4_renderer-boot §3（重连状态机运行流）、6_tui-cli-prompt-chain（TUI booting buffer + hydrate 六路、CLI reconcile 前缀比对）、5_resource-limits §2b（事件背压闭环）、3_idempotency（重试矩阵与幂等）各有其篇；本篇为**新增域**——首次把四把伞收口成断线自愈总图，并引入此前未覆盖的实码：页面生命周期四事件与 suspended-timer 坑（connection.ts:219-240）、三选一状态策略对照表（事件补发/失效重取/投影对账的适用面与代价）、watchdog 3× 心跳容差设计、`refetchType:"none"` 的削峰语义、LLM 依赖"无自动模型降级"的实测结论、retry-after 15min 封顶与超时 3 次专项预算、models.dev 四级降级链、contender 池的"错误不掩盖"纪律。v1 时代的重连是无状态的固定间隔重试；v2 的 generation/forced/lastActivity 三变量状态机是后来随着桌面端（pageLifecycle）与多实例（contender）需求长出来的。

## 相关篇

- [../5_timeline/4_renderer-boot.md](../5_timeline/4_renderer-boot.md) §3 — 重连状态机的运行流与装配位置（本篇 §1 补其时间轴与容差）
- [../5_timeline/6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md) — TUI booting buffer + hydrate 六路、CLI reconcile（本篇 §2 策略 C 的机制面）
- [5_resource-limits.md](5_resource-limits.md) §2b — 事件背压闭环（本篇 §4 只补被断者恢复路径）
- [3_idempotency.md](3_idempotency.md) — 重试矩阵与铸 id 幂等（本篇 §5.1 的对偶面）
- [../6_modules/14_mcp.md](../6_modules/14_mcp.md) — MCP 连接生命周期（本篇 §5.2 只收韧性口）
- [../6_modules/3_model-protocol.md](../6_modules/3_model-protocol.md) — 模型目录与出站管线（本篇 §5.3 只收降级链）
- [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5 — ensure 语义与 spawn 位置（本篇 §6 只收竞争纪律）
- [1_durability.md](1_durability.md) — 进程死亡的 durable 恢复（与本篇互补：那篇管"死了怎么活"，本篇管"活着但连不上"）
