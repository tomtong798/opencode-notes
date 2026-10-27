# 测试基建（Testing Infrastructure）

> **一句话**：用 1064 个测试文件 / 27.6 万行测试代码（测试:生产 ≈ **1.8:1**，2026-10-22 实测）组成五层防线——bun test 单元层、VCR 磁带层、JSON-RPC 模拟层、Playwright 浏览器层、CI 编排层——每层各自回答一类"改动怎么保证不坏"。
> **基线**：`/home/donghe/github/opencode` v2.0.30（dev 分支），行号实测 2026-10-22。
> **权威出处**：根 `AGENTS.md` Testing 节（:166-169）、`packages/app/AGENTS.md` Tests 节（:46-56，desktop/gui-extensions 声明遵循它）、`packages/ai/AGENTS.md` Tests 节（:16-20）、`packages/app/e2e/AGENTS.md`（Test Hygiene / Test Ownership）、`.opencode/skills/` 四个测试技能（unit-tests / tui-tests / app-e2e-tests / component-tests，共 477 行——把"什么让 CI 变慢变 flaky"的实战教训固化成了技能文档）。
> **互链**：工具本体介绍归 [../2_tech-stack/2.5_tooling/7_playwright.md](../2_tech-stack/2.5_tooling/7_playwright.md)、[8_storybook.md](../2_tech-stack/2.5_tooling/8_storybook.md)、[9_http-recorder.md](../2_tech-stack/2.5_tooling/9_http-recorder.md)、[3_oxlint.md](../2_tech-stack/2.5_tooling/3_oxlint.md)——本篇讲"仓库怎么把它们组成防线"；E2E 隔离根细节见 [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5.1。

---

## 0. 恐惧清单

| # | 恐惧 | 具体形态 | 防线（本篇章节） |
|---|---|---|---|
| F1 | **provider 行为漂移** | Anthropic 改了 tool-call 顺序校验、OpenAI 改了 SSE 格式，手写 mock 看不出真实请求形状 | §2 http-recorder 磁带 + §5 recorded 系列 |
| F2 | **UI 回归** | tabs/settings/review/terminal 改一处坏三处；视觉契约（像素、滚动锚定）无人看守 | §1 e2e keeper + §6 storybook/组件测试 |
| F3 | **并发竞态难复现** | 重连退避、TTL 过期、fiber 间 debounce，真实时钟下 20 次才复现一次 | §1.4 TestClock/ManualClock + tui-tests 技能 |
| F4 | **测试慢到没人跑** | Windows CI 上每个 git spawn 40ms、npm 测试一次 75 个 git 进程 | §1.4 时间预算 + §8 turbo 分包并行 |
| F5 | **测试互相污染** | 测试写了真实 HOME、真发了网络请求、磁带里漏了 API key | §4 隔离根 + §7 网络/秘密守卫 |
| F6 | **假测试**（永远绿的测试） | mock 了被断言的行为、断言被测代码自己产的值、source-grep 墓碑 | §1.3 纪律条款 + §7 guard |
| F7 | **无 LLM 的 UI 端到端** | 真调 LLM 的 UI 测试不确定、慢、贵 | §3 simulation 模拟协议 |
| F8 | **flaky 掩盖真回归** | "重试就过"让红灯失去公信力 | §9 flaky 治理（retry 上限 + Flaky 名单） |

根 AGENTS.md 把这些恐惧的操作结论压成两句（:168-169）：*测试不能从仓库根跑*（guard 见 §7）；*写测试前先读对应技能*——因为"什么让测试慢或 flaky"的知识散在几十次 CI 事故里，不读会重蹈覆辙。

这些恐惧不是假设。unit-tests 技能开篇给出的成本事实：CI 在 4 vCPU 的 Linux 和 Windows runner 上以 `--concurrency=3` 跑全部单测（三个套件分抢四个核），**Windows 永远是慢的那条腿**；tui-tests 给出：tui 套件在 CI 上 Linux 136-146s、Windows 147-171s，"most of it idle on frame pacing, production timers and sleeps"（大头耗在帧步进、生产定时器和 sleep 的空转上）。也就是说 1.8:1 的测试量要在一个 PR 的反馈周期内跑完，每一层都被"预算"倒逼设计。

---

## 1. 测试分层全景（含 1.8:1 实测数据）

### 1.1 文件数与规模（find 实测，2026-10-22）

```
opencode/packages/                       测试文件总量：1064 个 = 983 *.test.ts + 81 *.test.tsx
│                                                另有 110 *.spec.ts（Playwright）+ 83 *.stories.tsx
├── 单元层（bun test，*.test.ts/tsx，同包 test/ 目录或源码旁）
│   ├── core/      262 文件   ← 最大：session/bus/tool/config/mcp…几乎每个域都有专属测试文件
│   ├── tui/       172 文件   ← OpenTUI renderer 测试（tui-tests 技能管辖）
│   ├── app/       163 文件   ← 含 test-browser/ 40 个（浏览器环境单测）
│   ├── ai/        135 文件   ← 其中 56 个 *.recorded.test.ts（磁带回放，§2）
│   ├── cli/ 49  codemode/ 34  desktop/ 32  gui-extensions/ 31  server/ 29
│   ├── merman/ 23  client/ 21  session-ui/ 20  schema/ 15  util/ 13  sdk/ 11 …
│   └── simulation/ 9  http-recorder/ 4  …（测试基建自己也有测试，§10）
├── E2E 层（Playwright，*.spec.ts）
│   ├── app/e2e/regression/       42 spec  ← keeper 套件（§1.3）
│   ├── app/e2e/performance/      32 spec  ← 基准，CI 不跑（playwright.config.ts:38 testIgnore）
│   ├── app/e2e/user-story/        1 spec
│   ├── app/e2e/service-worker/    1 spec  ← 独立 config，随 shard 2 复用产物（test.yml:282）
│   ├── app/e2e/reproduction/timeline-suspense/ ← 独立 vite 工程的最小复现
│   ├── app/component-tests/      13 spec  ┐
│   └── session-ui/component-tests/ 21 spec ┘ Storybook Playwright（§6）
└── 行数：测试代码 27.6 万行 vs 生产 15.0 万行 → 1.8:1
```

### 1.2 分层对应关系

| 层 | 运行器 | 回答的问题 | 慢/贵在哪，怎么压 |
|---|---|---|---|
| unit（bun test） | `bun test --timeout 30000 --only-failures`（tui/ai/desktop 等）；core 经 `script/test.ts` 包一层（§4.2）；client/sdk/httpapi-codegen 用默认 5s | 模块契约 | 时间受控（TestClock/假定时器），Windows spawn 预算化 |
| TUI renderer | bun test + `@opentui/core/testing`（testRender/createTestRenderer/createAppFixture） | 终端 UI 帧级行为 | 帧数当预算不当时钟；30fps 测试帧率；一个进程跑全部 172 文件 |
| recorded（磁带） | bun test + http-recorder | provider 真实协议形状 | 零网络、零 key、可重放；按 RECORDED_* 环境变量选子集 |
| component | Playwright + Storybook dev server | 组件视觉/交互契约 | 每套件一个共享 server，CI 2 worker；后端全 in-page mock |
| e2e | Playwright + 生产构建 + mock server | 用户旅程 | 2 shard × 2 OS，只跑受影响 PR，dev 永不跑 |

五层之外还有两条非测试的守线：oxlint/ast-grep 的 lint（effect 简化规则自带测试，§7）与 turbo 的 `typecheck` 任务——它们与测试共同构成 CI 的完整红灯面，但本篇只收"会执行被测代码"的那部分。

### 1.3 测试纪律（AGENTS.md 条款摘要）

**app（:46-56，desktop/gui-extensions 均声明遵循）**——核心是"一个契约一个 owner"：

> "A test must pay for its upkeep… which observable contract does it protect, which credible regression makes it fail, and why does no existing test already catch that regression."（:48）

- One contract, one owner（:50）：一个行为只测一次，选最强边界——要么该域的 e2e keeper，要么属主模块的 unit；先扩表/keeper case 再开新文件。
- 不给生产代码开测试缝（:52）：禁止 test-only 导出、`now`/`delay`/`budget` 参数、`data-*`；时间用 `setSystemTime` 控制。
- 测试必须能失败（:53）：不许 mock 被断言的行为、不许软断言守护的顺序；回归测试必须在修复前的代码上失败。
- source-grep 禁令（:51）：不测源码文本（import/字符串 grep），**例外**是"文本本身就是契约"（如持久化的 key）。

**desktop（AGENTS.md:3-11）**——"表格化跨通道"：

> "Write them as tables: one table across channels, not one test per channel."（:6）

打包/安装器/IPC 校验/凭据/SQLite 迁移是契约；IPC 测试走 renderer 发出的原始消息格式而非 mock client（:8）；`bun typecheck` 不覆盖测试文件，改了哪个测试文件就 `bun test <file>` 跑哪个（:11）。

**e2e（packages/app/e2e/AGENTS.md）**——keeper 制 + 卫生条款：

- Test Ownership：每个产品域在 `regression/` 有一个 keeper 套件（tabs、settings、review、terminal、timeline history…），新回归 = keeper 里加 case，不是新 spec 文件；页面搭建一律走 `utils/` 共享 harness（app.ts/workspace.ts/mock-server.ts，含 `mockServers` 与 mock PTY），**不许在 spec 里手写路由/存储种子/fixture**；harness 缺什么就补进 utils 一次而不是到处复制；utils 下的文件不得叫 `.spec.ts/.test.ts`（否则 Playwright 会把它们当测试跑）。
- Test Hygiene（选摘）：用 role/label/text/test-contract locator，不许用 `.first()/.last()` 压 strictness 报错；**禁止 `waitForTimeout`/`setTimeout`/sleep/动画帧计数做同步**——等具体的 UI 状态、请求、事件；DOM 挂载或可见不等于异步渲染就绪，断言下一步动作真正需要的那个状态；事件/网络等待要注册在触发动作**之前**；状态改变的动作不重试（重试幂等的就绪检查，动作做一次、断言结果）；像素/CSS 值只在"测量即契约"时断言，且只在 owner keeper 里断一次；`performance/` 是 CI 不跑的基准，永远不用基准证明正确性。

**ai（AGENTS.md:16-20）**——磁带纪律：需要 layer 的测试用 `testEffect`（test/lib/effect.ts）；provider 测试默认回放磁带、无需 key；录制 = unset `CI` + `RECORD=true` + 该 provider 的 key；只重录要改的那盘（磁带含易变 ID，噪声 diff）。

**根 AGENTS.md Testing 节（:166-169）**：能不 mock 就不 mock（globalThis.\* 仅最后手段）；测真实实现、不把逻辑复制进测试；测试从包目录跑（见 §7 guard）；写测试前读对应技能。

### 1.4 时间与预算（tui-tests / unit-tests 技能的实证规则）

测试的快慢与稳定首先是一个**时间控制**问题。仓库的三条铁律：

- **Effect 代码用 TestClock**：`it.effect`/`testEffect`（core `test/lib/effect.ts:50-80`，`testEnv = Layer.mergeAll(TestConsole.layer, TestClock.layer())`）让 Effect 代码的时间由测试拨动——`advance`/`drain`（core `test/lib/clock.ts`）交替"1ms 真宏任务 + 500ms 虚拟步进"来跨 fiber 推进 debounce；由真实进程/socket 驱动的事件必须 `it.live`（TestClock 下真超时永不触发、缺事件变成挂死而非失败——AppProcess 测试的 1s post-exit 捕获期限就栽过这个）。**产品 Effect 代码必须读调用者的 Clock**（裸 `Effect.runPromise` 落到真实时钟是 bug，技能点名 `mcp/stdio.ts:66` 作为现存反例）。
- **非 Effect 代码用假定时器**：`jest.useFakeTimers()` + `setSystemTime`（bun:test），`afterEach` 恢复；推进用 `advanceTimersToNextTimer()` 而非小步循环（10ms 步进曾把 1000 种子 fuzz 测试变成 43 万轮事件循环：Windows 2.8-3.8s）。
- **帧数预算 ≠ 时间**：TUI 的 `waitForFrame`/`waitFor` 默认 20 passes（一 pass = 一帧，`@opentui/core/testing` 的 `DEFAULT_MAX_PASSES`）；"Timed out waiting for frame predicate after 20 passes" 是帧预算耗尽不是超时。测试渲染器 30fps、生产 60fps，所以预算天然折半。等具体信号而非等帧：焦点要等 `currentFocusedRenderable instanceof InputRenderable`（曾因 keys 丢失而非迟到 flake 5 次/20 小时，加大预算到 150 passes 也没用）；markdown 要等 tree-sitter 高亮（`waitForHighlightedFrame` 在谓词内 await 所有 in-flight highlight，不耗 passes）；无信号可等才用墙钟 deadline 轮询。
- **Windows 成本模型**："按 spawn 计数估 Windows 风险，不按本地墙钟"——npm.test.ts 一个 Git 安装 case 本地 0.4s、Windows CI 撞 5s 上限（约 75 个 git 进程，git spawn ≈40ms）；core 因此把 runner 默认调到 30s（core `script/test.ts:38` 注释）。超时只在"超过包默认"时才放宽预算，且要注释说明正当工作（如 server `fetch.test.ts:406-407` 的真重试 18×200ms）。
- **真轮询用截止时间不用次数**：`until`（30s）/`eventually`（10s）毫秒级 generous deadline，满足即返回；100×10ms 的次数循环在负载下会超（config/reload.test.ts 的 100×10ms 曾在 Linux dev push 上 1.2s 超时）。禁止用更大 timeout/`--retry`/`skipIf(win32)` 吸收 sleep——删掉等待本身。

---

## 2. http-recorder 磁带

### 2.1 是什么

`packages/http-recorder`（自研 ~1600 行）：VCR 式 HTTP/WebSocket 录制回放。首跑真调 API 并把整个交互序列写成 JSON 磁带（cassette），之后所有运行回放磁带——**确定性、离线、零费用**。README 的 mermaid 图即决策流：有磁带→回放；无磁带且本地→录制；**无磁带且 CI=true→直接失败**（`src/http/recorder.ts:144` 的报错原文："Run locally to record it (CI=true forces replay)."）。

录制/回放的切换**不需要改任何代码**：`HttpRecorder.layerFetch(name)` 返回一个 Effect Layer 替换 HttpClient 传输层——应用代码（含 Effect 4 beta 的 HTTP/WS 栈）不知道响应是活的还是磁带。这个"传输层替换而非请求拦截"的实现位置也是它与 nock 等拦截库的本质差异：磁带坐在 Effect 的 service 图里，类型系统保证没有旁路。

### 2.2 API 与磁带格式

```
src/index.ts:10-26  暴露四个入口：
  layerFetch(name, opts)               → 替换 Effect HttpClient（HTTP 磁带）
  layerSocket(name, opts)              → Effect Socket（WS 磁带）
  layerWebSocketConstructor(name, …)   → 浏览器式 WebSocket 构造器
  hasCassetteSync / removeCassetteSync → 存在性检查 / 重录前删旧带
```

磁带格式（实测 `core/test/fixtures/recordings/session-runner/openai-chat-streams-text.json`）：

```json
{ "version": 1,
  "metadata": { "name": "session-runner/openai-chat-streams-text", "recordedAt": "2026-09-02T…" },
  "interactions": [ { "transport": "http",
      "request":  { "method": "POST", "url": "https://api.openai.com/v1/chat/completions",
                    "headers": {…}, "body": "{…}" },
      "response": { "status": 200, "headers": {…}, "body": "…" } } ] }
```

默认目录 `test/fixtures/recordings/`（`src/cassette/store.ts:4` `DEFAULT_RECORDINGS_DIR`）。回放按请求匹配逐条 `claim`（`src/replay/state.ts`：SynchronizedRef 记已用 index，防止同磁带交互被错误复用），不匹配即失败并给出差异详情（`http/recorder.ts:134-146`）。

### 2.3 两道安全阀（磁带层的"测试的测试"）

- **秘密扫描**：写盘/回放前对磁带全量跑 `secretFindings`（`src/redaction/secrets.ts`）——正则模式（sk-…、Bearer 等）+ **与环境变量的比对**（`ENV_SECRET_NAMES` 命中的 env 值若出现在磁带里即失败）；发现即抛 `UnsafeCassetteError`（`store.ts:26`），写路径 :86 直接 fail。这就是 ai 包敢用 `"fixture"` 占位 API key 的底气。
- **CI 禁录**：录制模式要求 unset `CI`——ai 的封装在 `RECORD=true` 且 `CI !== undefined` 时直接 throw（`recorded-test.ts:90`）。

### 2.4 provider 测试怎么用它（ai 包 recorded 体系）

`packages/ai/test/` 有一套 881 行的录制框架：

```
recorded-test.ts    (118) 对外入口 recordedTests()：组装 LLM/Image/Video/Speech/Transcription/
                         Evaluation 客户端 layer + HTTP + WS 磁带（:95-117），fetch 与 WS 双轨都进磁带
recorded-runner.ts  (101) 通用执行器：算 cassette 名（prefix/kebab-case，recorded-utils.ts:54）、
                         查重（Duplicate 报错 :74）、RECORD 开关、无磁带→test.skip（:83-85）、
                         有 requires 缺 key→录制时 skip（:78-81）
recorded-utils.ts    (56) 选择器：RECORDED_PREFIX / RECORDED_PROVIDER / RECORDED_TAGS /
                         RECORDED_TEST 四个环境变量过滤子集（:38-41）
recorded-scenarios.ts(502) 跨 provider 复用的标准请求场景（weatherTool 等）
```

消费规模实测：**56 个 `provider/*.recorded.test.ts`**、`fixtures/recordings/` 下 **274 盘磁带**（openai-responses/anthropic-messages/bedrock/google-vertex/mistral…）。回放是默认态且不需要任何 key（ai/AGENTS.md:19）；重录只重录正在改的那一盘——磁带含易变 ID，全量重录 diff 噪声大（:20）。

一个典型 recorded spec 的形状（`anthropic-messages.recorded.test.ts`）：模型配置 `apiKey: process.env.ANTHROPIC_API_KEY ?? "fixture"`（回放时占位即可）；`recordedTests({ prefix, provider, protocol, requires: ["ANTHROPIC_API_KEY"], options: { redact: { allowRequestHeaders: ["anthropic-version"] } } } })` 一行声明 provider 身份、录制所需凭据和**磁带里允许保留的请求头**（redact 策略逐 case 可调）；测试体就是普通 Effect 断言（如对畸形 tool-call 顺序 `Effect.flip` 拿 AIError）。个案可 `.with(name, { cassette, tags })` 覆盖磁带名/打标签（tool、sad-path…），RECORDED_TAGS=tool 即可只跑工具类回放。

---

## 3. simulation 模拟协议

### 3.1 解决什么问题

F7：想端到端测"UI 对 LLM 流的响应"，但真 LLM 不确定、慢、贵。`packages/simulation`（~2200 行）给出答案：**双进程 JSON-RPC 模拟协议**——前端与后端各挂一个 loopback WebSocket 控制端点，**驱动方（测试/agent）从外面同时操纵两侧**：

```
       驱动方（测试代码 / AI agent）
        │ ui.type / ui.click / ui.snapshot      │ llm.chunk / llm.finish
        │ （WebSocket→前端控制服务器）            │ （WebSocket→后端 SimulatedProvider）
        ▼                                       ▼
┌─ TUI 前端（Drive renderer，可 headless）─┐   ┌─ opencode server（--simulate）────────┐
│ simulation/frontend/                     │   │ simulation/backend/                    │
│  server.ts   ui.* RPC                    │   │  simulated-provider.ts (918行)         │
│  actions.ts  type/click→真键盘/鼠标事件   │   │   llm.chunk→textDelta/toolInputDelta   │
│  renderer.ts headless 渲染 + 语义快照     │   │  openai.ts  事件流编码成 OpenAI SSE     │
│  semantics.ts role/selected 语义标注      │   │  network.ts 全出站 HTTP 走模拟路由表，  │
└──────────────────────────────────────────┘   │   未知目的地一律拒绝                    │
                                               └────────────────────────────────────────┘
```

协议本体在 `packages/protocol/src/simulation.ts`（691 行，Schema 定义）：前端能力 15 个（:199-216 `ui.type/press/enter/arrow/focus/click/click.semantic/mouse/recording.pointer/resize/matches/state/snapshot/capture/recording.finish`），后端能力 `llm.attach/chunk/finish/disconnect/pending/request/tool-input-delta`、`tool.attach/update/finish/fail`（:436-445）。`ui.snapshot` 返回 `opencode-ui-snapshot-v1` 格式的语义树（:315），`ui.click.semantic` 按 role/label 点——**断言的是语义身份而非像素坐标**（semantics.ts 注释："ui.snapshot expose the same identity"）。

两侧 RPC 的职责分工（协议请求定义 :666-687）：

```
方法（JSON-RPC request）     谁实现   语义
─────────────────────────────────────────────────────────────
ui.snapshot                  前端     全量语义树（v1 格式，role/label/selected…）
ui.type / ui.click / ui.press/enter/arrow/focus        前端   输入注入
ui.click.semantic            前端     按 role+label 定位并点击（抗布局变化）
ui.matches                   前端     谓词式断言（等 UI 进入某状态）
ui.state / ui.capture / ui.resize / ui.recording.*     前端   状态/截图/尺寸/指针录制
llm.request / llm.attach     后端     声明一次 LLM 调用 / 挂接到某调用
llm.chunk / llm.finish       后端     注入一条流事件（textDelta 等）/ 结束（带 reason）
llm.tool-input-delta / llm.pending / llm.disconnect    后端   工具参数流 / 挂起 / 断开
tool.attach / tool.update / tool.finish / tool.fail    后端   工具生命周期事件注入
```

握手（Handshake）双向声明能力，驱动方据 capabilities 决定用 `ui.click` 还是 `ui.click.semantic`——协议自身版本化（`decodeRequestEffect` 用 Schema 严格解码每个入站请求，坏的 JSON-RPC 在协议层就拒绝）。

### 3.2 接线与消费方

- 后端入口：`OPENCODE_SIMULATE=1` → cli `server-process.ts:106` 置 `simulation: true` → server `routes.ts:153-158` 动态 import `simulationReplacements` 把 app 的 httpClient/provider 节点换成模拟实现（backend/index.ts:20-26 注释：所有出站 HTTP 解析到模拟路由表、未知目的地拒绝、ModelsDev 目录请求用空目录应答）。
- 前端入口：tui `app.tsx:260` 动态 import `Drive.create`；`OPENCODE_DRIVE_RENDERER=headless` 时用 SimulationRenderer 无头渲染（frontend/simulation.ts:10），manifest（`manifest.ts`）校验端点必须是 `ws://127.0.0.1:<port>`。
- **消费方**：① simulation 自身的 9 个测试（协议编解码、control-server、actions、`openai.test.ts` 验证模拟事件逐字编码成 OpenAI SSE、`simulated-provider.test.ts` 759 行用真 core 层跑 Drive 控制的 provider）；② tui `test/simulation/semantics.test.ts` 验证生产组件的语义标注与模拟渲染器共享同一 lazy reader（`Symbol.for("opencode.simulation.semantics")`）；③ 名字里的 "Drive" 暗示最终用途——让 AI agent 驱动真实 UI 做探索性测试。

与 http-recorder 的分工：磁带把**过去某次真实的 HTTP 往返**固化重放，锁协议形状；simulation 把 **LLM 的产出**换成驱动方即时注入的确定事件，锁 UI 行为。前者回答"我们的请求/解析对不对"，后者回答"UI 对这种流会怎么表现"——两层都不需要真 LLM。

---

## 4. E2E 隔离根

### 4.1 desktop 的 createTestRoot

`packages/desktop/src/main/lifecycle/configure.ts:52-73`（详见 [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5.1）：

```ts
function createTestRoot() {                                    // :52
  const root = testOnboarding                                  // :9 ← OPENCODE_TEST_ONBOARDING === "1"
    ? path.join(tmpdir(), `opencode-onboarding-${randomUUID()}`)   // 每次全新 UUID 目录
    : app.isPackaged ? undefined
    : process.env.OPENCODE_DESKTOP_TEST_ROOT                   // 未打包 + 显式环境变量
  if (!root) return undefined
  if (testOnboarding) rmSync(root, { recursive: true, force: true })
  for (const dir of ["data","config","cache","state","desktop","session","documents"])
    mkdirSync(path.join(root, dir), { recursive: true })
  if (testOnboarding) process.env.OPENCODE_DB = ":memory:"     // :66 ← DB 整个进内存
  process.env.XDG_DATA_HOME   = path.join(root, "data")        // XDG 四件套全重定向
  process.env.XDG_CONFIG_HOME = …  XDG_CACHE_HOME = …  XDG_STATE_HOME = …
  return root
}
```

配合 `app.setPath("userData"/"sessionData"/"documents")` 重定向 Electron 自身路径（:32-39）：**一个环境变量把 Electron + opencode + XDG 三层状态全部圈进临时目录**，测试进程与开发者真实机器互不可见。`OPENCODE_TEST_ONBOARDING` 还被 `electron.vite.config.test.ts:18-31` 用来断言"onboarding 预览只在 dev serve 注入 renderer"这类构建契约。

### 4.2 core 的环境擦洗（同样的思想在 CLI 侧）

core 的 `bun test` 不是直接跑，而是经 `script/test.ts` 包一层子进程，环境先擦洗（:1-40）：HOME/OPENCODE_TEST_HOME/XDG 四件套指向新 tmpdir；用正则剥掉所有 `*_API_KEY/AUTHORIZATION/TOKEN/SECRET/PASSWORD/CREDENTIALS` 类 env（`RECORD=true` 时只放行 OPENAI_API_KEY）；Windows 再加 USERPROFILE/TMP/TEMP。**效果：core 的 262 个测试文件在同一进程里共享"干净但真实"的 HOME**，既不碰开发者配置也不互相污染。

### 4.3 app e2e 的环境准备

- 每测试 fresh context（无 beforeAll/serial/worker fixture，`fullyParallel` 可随时开）。
- 无真后端：一个 Bun 静态服务器（`e2e/utils/serve.ts`，刻意替代 vite preview——后者在不区分大小写文件系统上每请求同步读 3100 个资产的目录）+ **进程内 mock server**（`e2e/utils/mock-server.ts:202` `mockOpenCodeServer`：Effect HttpApi handler 应答 `page.route("**/api/**")`、init-script 伪造 `/api/event` SSE、`routeWebSocket` 供 PTY）。
- **未 mock 的路由直接失败**：mock 未定义的路由返回 501 并 throw "Unmocked API request: … (add it to e2e/utils/mock-server.ts)"（mock-server.ts:403-411）——宁可红也不许请求漏到 SPA fallback 拿个 HTML 200 假绿。
- 时钟：测试默认真时钟，需要时用 Playwright 的 `page.clock`（必须在触发定时器的 goto/reload **之前**安装）；mock 目录路径统一用 `C:/OpenCode/<name>`（workspace.ts:33）使 Windows 失败可复现为路径无关的竞态而非平台代码路径。
- CI 细节：生产构建（`PLAYWRIGHT_BUILD=1`）、CI 5 workers、retry 2、首失败无 trace/video（retry1 才有）——所以 flaky 调查流程是固定三步：error-context.md（失败的 aria 快照）→ retry trace 时间线 → 逐帧抽 video。

---

## 5. 录制回放的会话测试

core 不止有 ai 层的 provider 磁带，还有**把整条 session 执行链接上磁带**的集成测试——`packages/core/test/session-runner-recorded.test.ts`（248 行）：

- :49-54 定位磁带 `session-runner/openai-chat-streams-text`；`RECORD=true` 且 CI 有值 → throw（:52）；录制前 `removeCassetteSync` 删旧带。
- :55-56 用 `HttpRecorder.layerFetch` 喂给 `RequestExecutor`，再叠 `LLMClient.layer`——**从 HTTP 往上整条 LLM 栈都是生产代码，只有最底下的传输层是磁带**。
- :59-70 模型配置指向真 endpoint（`api.openai.com/v1`，key 用 `?? "fixture"` 占位），`temperature: 0` 保证录制请求确定。
- 其余依赖（instructions/skills/config/plugin supervisor）逐个 `Layer.mock` 或 replace 成空实现——测的是 SessionRunnerLLM 把流式 HTTP 响应变成正确消息序列的**主干**，不是外围。
- 断言（:64-70 附近）：两条消息（user prompt 原文 + assistant finish:"stop"），assistant 内容匹配磁带里的流式文本。

同族的 recorded 思想散布 core 各处（session-prompt/session-instructions/session-diff 等测试文件引用 fixture 体系）；ai 侧另有 `transcription-recording.ts`、`recorded-golden.ts`（金样比对）。

这套测试的价值定位：介于"全 mock 单测"与"真 LLM 集成测试"之间——

```
全 mock 单测        recorded 会话测试            真 LLM 集成
（ai/test/llm.test.ts）  （session-runner-recorded）   （CI 里不存在）
假 route + 假流  →  真 HTTP 字节 + 真解码/投影栈  →  不确定、慢、贵
快但看不见协议形状     锁住协议形状 + 消息管线          （被 §2 磁带替代）
```

---

## 6. TUI Storybook

两个"storybook"，别混淆：

### 6.1 TUI 故事（fixture 驱动，无浏览器）

`packages/tui/src/feature-plugins/system/storybook/`——根 AGENTS.md "TUI Stories" 节立规：**渲染真实生产组件而非视觉复制品**；提交等副作用局限在 story 内、保证反复探索安全；状态维度暴露成 story 快捷键并列进 `StoryFooter`；组合可能把 fixture 带沟里时要提供 reset 命令；布局相关的 story 要在窄/宽两种终端尺寸下都过一遍。运行方式：`OPENCODE_STORY=<story-id> bun run dev:live`（app.tsx:343-352 把它变成初始路由插件；`OPENCODE_STORY=1` 开索引页）。fixture 机制实测：`one-cell-spinner.fixtures.ts` 用 `heldFrames`/`pulse` 纯函数生成帧序列——seed 呼吸/余烬等 spinner 的每一帧、间隔、亮度包络都是**代码生成的确定性数据**（`0.3 + 0.7 * level(phase)` 保底不灭，注释原话："a working indicator must never blink out entirely"——连 fixture 都携带产品约束）。当前 7 个 story（merman-layouts/session-tabs/session-location-missing/one-cell-spinner/link-clicks/highlights）。

它是"人肉探索回路"而非自动断言：改 spinner/theme/布局时先在 story 里穷举状态维度，比在生产里造出同样状态快一个量级——与 §6.2 的自动组件测试互补。

### 6.2 Web 组件测试（Storybook + Playwright）

`packages/storybook` 是共享 harness：`playwright/config.ts` 的 `componentConfig(directory)` 给 app/session-ui 各生成一份配置——一个 Storybook/Vite dev server 全 worker 共享（:17-24 webServer），CI 2 workers + retry 2；`playwright/story.ts` 提供 `mount(id, {args, globals})` fixture（整页 iframe 导航）与 `sourceURL()`（:5-9 注释警告：URL 拼写错误会加载**第二个模块实例**，其 context 页面不提供）。后端全 mock（viteFinal 的 resolve.alias 指向 `.storybook/mocks/app/**`）。spec 落在 `app/component-tests/`（13）与 `session-ui/component-tests/`（21）；故事文件与生产组件同目录（`*.stories.tsx`，全仓 83 个，索引 glob 由 `.storybook/main.ts:44-49` 控制）；"契约在 component-tests 证明过就不再写 app e2e 副本"（app AGENTS.md:56）。

组件测试层的已知脆弱点（component-tests 技能实证）：共享 dev server 的**依赖优化器**会在测试中途触发整页 reload（"optimized dependencies changed"）；URL args 会被 Storybook 静默丢弃（路径/带点的文件名不满足安全字符集）导致"测的是默认场景还绿着"；冷启动第一个 story 最慢。对策都已进 harness（args 必须有类型化默认值、`sourceURL` 字符串字面量、`onCleanup` 恢复 mock state）。

---

## 7. 守护性测试

仓库里有一类测试不测产品功能，**测仓库自己的纪律**：

| guard | 位置 | 机制 |
|---|---|---|
| **根目录禁跑测试** | 根 `package.json:36` `"test": "echo 'do not run tests from root' && exit 1"` | 根 AGENTS.md:168 引用其 guard 名 `do-not-run-tests-from-root`；测试必须从包目录跑（隔离根见 §4）——在根跑会命中依赖包的 node_modules 与真实环境，全部隔离设计作废 |
| **测试网络拒绝** | core `test/lib/effect.ts:18-29` `refuseNetwork` | 包一层 fetch：loopback 放行，其余 reject 并记 violation，`testEffect` 结束时把 violations 变成测试失败（:45）——防"测试悄悄打了真网络"；由 `test/network-guard.test.ts`（73 行）元测试其行为，并验证显式 HttpClient 替换不经过它（:30-46 注释点名 ModelsDev 这类吞请求失败的调用方也会被 violations 抓住） |
| **磁带秘密扫描** | http-recorder `redaction/secrets.ts`（§2.3） | 磁带含 env 秘密/已知 key 模式 → `UnsafeCassetteError` |
| **mock 覆盖强制** | app e2e `mock-server.ts:403-411` | 未 mock 路由 501 + throw（§4.3） |
| **effect 简化规则** | `script/ast-grep/effect-simplifications/`（rules + rule-tests 快照） | `test:effect-simplification-rules` 在 CI Linux leg 先跑（test.yml:99-104）：ast-grep 规则本身带测试，防止 lint 规则漂移 |
| **重复磁带** | ai `recorded-runner.ts:74` | 同名 cassette 二次注册 → Duplicate 报错，防止两个测试悄悄共享一盘磁带互相消耗 interactions |
| **utils 文件名守卫** | app e2e AGENTS.md | `utils/` 下文件不得以 `.spec.ts/.test.ts` 结尾，否则被 Playwright 当测试执行 |

source-grep 禁令（app AGENTS.md:51"不测源码文本"）与 desktop 的"No source-grep or tombstone tests"（desktop AGENTS.md:9）是同一原则：**测试断言行为，不断言代码长相**；例外仅当文本即契约（持久化 key 等）。

---

## 8. CI 编排

`.github/workflows/test.yml`（实测，无容器，全部 bare runner + Bun/Node）：

```
push(dev)/PR/workflow_dispatch
│
├─ job: affected   （blacksmith-4vcpu-ubuntu；turbo ls --affected 判定 @opencode/app、
│                   @opencode/storybook 是否受影响 → 决定 e2e 是否跑；dev push 永不跑 e2e）
│
├─ job: unit  ×{linux, windows}   fail-fast:false
│    ├ bun turbo test --affected --concurrency=3   （4 vCPU，3 并发包；GITHUB_ACTIONS=false）
│    ├ Effect 简化规则测试 + lint（Linux）
│    ├ codemode 发布 dry-run / workerd SDK verify:package（Linux）
│    └ 生成物校验：client check:generated、www 文档 check:generated（Linux）
│
├─ job: build ×{linux, windows}
│    ├ cli 单文件构建 + service-smoke（编译产物生命周命验证）
│    ├ 工作区库构建 turbo build --filter='@opencode/cli^...'（测试从不等依赖构建——turbo.json:38-40：
│    │   workspace 包直接导出源码，transit 任务把依赖折进 hash；build job 专门补上这条检查）
│    └ Node 运行时构建验证（node 26.4 + --node smoke）
│
├─ job: e2e-suite ×{linux, windows} ×{app-1, app-2, session-ui-components, app-components}（needs affected）
│    ├ app-N：bun --cwd packages/app test:e2e:built --shard=N/2   （生产构建，15min 超时）
│    ├ shard 2/2 顺带 service-worker 套件（复用其生产构建）
│    ├ 两个组件套件各占一条 leg（Storybook server 与时长不叠加）
│    ├ Playwright 浏览器按版本缓存；Node 锁 24.15（1.59 在 24.16 挂）
│    └ artifacts 上传（trace/video/report，7 天）
│
└─ job: e2e ×{linux, windows}   ← 聚合检查：各 OS 的分支保护只看这两个名字，
                                 任一 leg 失败则两 OS 判红（:316-335 注释明说这是设计）
```

关键编排决策：**shard 按 spec 文件数切**（注释 :219-221：Playwright 整文件分 shard，所有 app-N 必须同一 total）；**dev 分支永不跑 e2e**（affected 输出显式排除，红旗不污染默认分支历史）；并发组 cancel-in-progress 只对 PR 生效。另有独立 `storybook.yml`（仅 storybook/ui/session-ui 路径触发，构建静态 Storybook）与 `review.yml`（AI review，非测试）。

时间预算（从 workflow 实测）：unit 20min 上限、e2e 每 shard 15min、service-worker 5min、sdk 打包验证 15min、构建 10-15min。tui 全套在 CI 4 vCPU 上 136-171s（tui-tests 技能实测）——**一个 bun test 进程跑 172 个文件**，慢的大头不是断言是帧步进与产物等待。

turbo 编排的一个关键设计（turbo.json:38-40 注释）：workspace 包直接导出源码，**测试从不等依赖构建**——`transit` 任务不跑任何东西、只把每个工作区依赖折进测试 hash；库能不能构建由 build job 单独把关。这换来了 unit job 的高并行度（三个包同时测互不阻塞）。

---

## 9. 失效场景（测试自身的脆弱）

| 失效模式 | 症状 | 仓库的对策 |
|---|---|---|
| **磁带过期** | provider 改协议 → 回放请求不再匹配 → "Fixture does not match the current request" | 匹配失败**报差异详情**而非静默（recorder.ts:140）；重录需 unset CI + 真 key + `RECORD=true`，且只重录改动的那盘（易变 ID 噪声）；磁带 `version` 字段留格式演进空间 |
| **磁带泄密** | 录制时真 key 落盘 | 双重闸：录制要求 unset CI；写盘前 secretFindings 扫（env 比对 + 模式） |
| **磁带掩盖现网漂移** | 磁带永远绿，真 API 早已变了 | 这是**接受的权衡**：磁带锁的是"协议形状契约"，形状变了测试就该红；补录即显式承认契约变更进 review |
| **磁带易变 ID 噪声** | 重录一盘 diff 巨大，review 看不出语义变化 | 只重录改动的那一盘（ai/AGENTS.md:20）；cassette 名含场景 slug 便于对位 |
| **Storybook 依赖优化器中途 reload** | "optimized dependencies changed. reloading" → "Execution context is destroyed" | harness 吸收（config.ts stdout: "pipe" 保留日志供诊断）；fixture 模块走 `sourceURL` 字面量纳入预打包 |
| **flaky e2e（Windows）** | 重试后过，红灯公信力受损 | CI retry 2 上限；每次 retry 留 trace+video；报告 Flaky 名单要人工检查（app-e2e-tests 技能 "When CI flakes" 节：先读 error-context.md 的 aria 快照，再看 retry trace 的时间线、逐帧抽 video）；技能记录的实证：调查过的 8 个 flake 里 3 个是**真产品 bug**（已修） |
| **帧预算当时钟** | TUI 测试用 maxPasses 数帧数覆盖 1s 延迟 → 帧率一变全挂 | tui-tests 技能明令禁止，列了现存违规清单（app-lifecycle:792 等）——技术债显式记账而非假装没有 |
| **测试互相污染** | 文件间泄漏 state/端口 | core 每次运行全新 HOME；TUI fixture server 一律 `port: 0`（Windows 49152+ 临时端口可能被占）；组件测试 `onCleanup` 恢复 mock 状态 |
| **假绿**（mock 了断言对象） | 测试永远过 | AGENTS.md 纪律条款（§1.3）+ "回归测试必须在修复前失败"的可操作标准 |
| **慢测试** | 没人本地跑 → 问题晚发现 | 时间控制规则（§1.4）+ Windows spawn 预算 + turbo affected 让 PR 只跑受影响包 |

---

## 10. 元覆盖（测试的测试）

本篇自指：防线自己也要被看守。实测清单——

- **http-recorder**：自身 4 个测试文件（cassette/http/redaction/websocket），磁带读写、请求匹配、秘密扫描各直接测；`test/support.ts` 提供共享脚手架。
- **simulation**：9 个测试覆盖协议编解码（protocol.test.ts 断言 `ui.snapshot` 等 RPC 可解码）、control-server 握手与能力协商（frontend-server.test.ts:42 断言能力数组含 ui.state/ui.snapshot/ui.click.semantic/ui.capture）、SSE 编码逐字节断言（openai.test.ts:7-27 把 textDelta/finish 流的期望 SSE 帧逐行写死）、759 行的 simulated-provider 全链路（用真 core 的 Bus/Database/Session 层跑 Drive 控制的 provider 调用并验证 finished invocation 被清理）。
- **core 测试 harness**：`network-guard.test.ts` 元测试 refuseNetwork（§7）；`test/lib/effect.ts` 的 TestClock/TestConsole 组装被三个包共用（server/cli 从 core import）。
- **lint 规则**：ast-grep 规则带 rule-tests 快照，CI 先 `test:effect-simplification-rules` 再 `lint:effect-simplifications`——**规则失效会在测试层先红**。
- **磁带元数据**：每盘磁带带 name/recordedAt/tags metadata（recorded-runner 注入），`hasCassetteSync` 决定 skip 与否——磁带库存本身可审计。
- **Playwright 配置**：`test-browser/playwright-config.test.ts` 校验 config 变更（app-e2e-tests 技能 :121）。
- **构建契约**：electron.vite.config.test.ts 用环境变量开关断言 define 注入（§4.1）。
- **共享 harness 自身有契约测试**：e2e `typecheck:e2e` 允许存量 4 个错误（文件:行号在技能里列死），新增即 review 红灯——债务基线显式化。

元覆盖的共同形态：**每道防线在首跑时就为"防线坏了"准备了一个会红的测试**。refuseNetwork 有 network-guard、秘密扫描有 redaction.test、SSE 编码有逐字节断言、lint 规则有快照——这与"回归测试必须在修复前失败"（§1.3）是同一条纪律的两次应用。

---

## 11. 演进注记（旧笔记没有的）

- **测试知识技能化**：`.opencode/skills/` 下四个测试技能（unit-tests 140 行 / tui-tests 82 / app-e2e-tests 138 / component-tests 117）是本仓库独有的机制——把"什么让 CI 变慢变 flaky"的实证教训（具体到文件:行号和失败次数："npm.test.ts 在 45 次 Windows core 运行中 13 次撞 5s"、"dialog 等待缺焦点谓词时 20 小时内 5 次 CI flake"）写成**给 AI/新人的必读前置**，根 AGENTS.md:169 强制引用。传统的"测试规范"是禁止事项清单，这里是**带事故复盘的案例库**。
- **Drive/simulation 双向化**：simulation 包不只是测试工具——`Drive` 命名、headless renderer、manifest 校验、`recording.ts` 的 timeline 回放，指向"录制操作序列 → 确定性重放 → AI agent 驱动 UI"的产品化路径（`/recording`、`/backend`、`/frontend` 三个导出边界）。
- **磁带安全的 env 比对**：http-recorder 不只做模式匹配，还把**当前进程环境变量里的可疑值**逐一与磁带内容比对（secrets.ts:31-37）——录制机的环境本身被当作泄密源。这在通用 VCR 库（Ruby VCR/Polly.js）里不常见。
- **Playwright 版本与 Node 的锁死注释**（test.yml:244-246）："Playwright 1.59 hangs while extracting Chromium with Node 24.16"——工具链互斥靠注释 + pin 双 24.15 看守，提醒升级时这里有雷。
- **e2e 聚合检查的分支保护设计**（test.yml:316-335）：`e2e (linux)`/`e2e (windows)` 两个必要检查只是四个 leg 的"与门"，任一 leg 失败两 OS 全红——防止"leg 名字太多、分支保护漏配一个就放行"的组织学失效。

---

## 相关篇

- [../2_tech-stack/2.5_tooling/9_http-recorder.md](../2_tech-stack/2.5_tooling/9_http-recorder.md) / [7_playwright.md](../2_tech-stack/2.5_tooling/7_playwright.md) / [8_storybook.md](../2_tech-stack/2.5_tooling/8_storybook.md)：工具本体 vs 本篇的"防线组装"
- [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5.1：createTestRoot 在启动链中的位置
- [1_durability]…[10_lifecycle]（本卷各篇）：每篇末节"测试覆盖"即本篇某层防线的产品域投影
- 仓库内一手资料：`.opencode/skills/{unit-tests,tui-tests,app-e2e-tests,component-tests}/SKILL.md`、`packages/*/AGENTS.md` Tests 节、`.github/workflows/test.yml`
