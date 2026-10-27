# V1 膨胀期：437 个文件的巨石与修不动的 TUI（2025-07~2026-03）

> 基线：git + appendix-data.md §4 | 实证为主，推断处标注（分析）
> 复现前提：仓库本地全史，`cd /home/donghe/github/opencode` 后逐条执行文中命令。
> 取证时点 commit（2026-04-01 前最后一个）：`c8ecd640`（2026-03-31），下文简称 **$PEAK**。

## 0. 时期速览

```
时间轴（V1 膨胀期，横向为月份）
2025-07    08    09    10    11    12    2026-01    02    03
  |--------|-----|-----|-----|-----|-----|---------|-----|----|
  #811     solid│   app   OpenTUI  sdk v2          #11762  effect
  渲染慢   TUI │   包诞生  替换 ink global bus      Ink提案  migration
  (💬84)   定型 │   09-15  10-31   12-07  12-10    02-02    plan 03-12
               │   desktop          │               │        Bus迁Effect
               │   10-03            │               │        03-18
               └─ #631 Windows 长跑贯穿全程（16 个月，2026-09 才终章）

src 文件数（实测 git ls-tree -r）：
117 ──────────► 195 ────► 332 ──────────► 437（峰值）
2025-08        11-01     2026-02-01      2026-04-01
                                          ╕ 2026-07-20 V1 主入口删除

月提交量：2026-01 = 2083（仓库史上第一峰）＝膨胀顶点
```

这一时期的三个关键词：**单包膨胀**（437 文件、43 个顶层目录全在 `packages/opencode/src` 一个包里）、
**修不动的 TUI**（tui 相关 fix 159 条，scroll/render 类修了又坏跨月复发）、**Effect 化两难**
（想换运行时基座，但巨石无法一刀切，只能先织安全网再动手术）。
2026-04-15/25 server/core 包诞生是下一卷的事——本卷讲的是"为什么必须动刀"。

## 1. 膨胀的解剖（单包巨石证据）

在峰值时点做一次"体检"：

```bash
PEAK=$(git log --format="%H" --until=2026-04-01 -1)
git ls-tree -r --name-only $PEAK -- packages/opencode/src | wc -l   # 437
git ls-tree --name-only $PEAK -- packages/opencode/src/ | wc -l     # 43 个顶层条目
```

43 个顶层条目 = 42 个目录 + `index.ts`。一个包里同时住着：

```
packages/opencode/src/（2026-04-01，$PEAK，目录: 文件数）
├── cli/        161  ← TUI（cmd/tui/ 124 个 .tsx）、CLI 命令、启动器
├── tool/        45  ← 工具子系统（bash/edit/grep/lsp-*...）
├── util/        35  ← 杂物抽屉
├── provider/    32  ← 模型提供商 + auth + sdk
├── session/     31  ← 会话引擎（V1 心脏）
├── server/      22  ← HTTP API（Hono）
├── storage/ 7  project/ 7  plugin/ 7  control-plane/ 7
├── effect/ 6  file/ 6  config/ 6  agent/ 6              ← Effect 器官已开始体内生长
├── lsp/ 5  sync/ 4  permission/ 4  mcp/ 4  acp/ 4  account/ 4
├── bus/ 3  command/ 3 ...
└── index.ts (1)  ── 单入口导出以上全部
```

职责混杂度：**引擎（session/tool/provider/lsp）、界面（cli/tui 的 124 个 tsx）、
协议出口（server/acp）、基础设施（bus/storage/effect）四层全部编进同一个包、
同一份 `index.ts` 导出表。**（实测：`cli` 一个目录就占总文件数 37%。）

对比 V1 起点（2025-05-30，TS 重写完成时 `f3da7355`）：当时 src 只有
**13 个目录、38 个文件**（app/bus/config/lsp/provider/server/session/share/storage/tool/util...），
每个目录一两把交椅，职责清爽。九个月里文件数 ×11，目录数 ×3。

膨胀的斜率（cli 目录独占大头，实测）：

```
            2025-08-01  2025-11-01  2026-02-01  2026-04-01
src 总文件      117        195        332         437
其中 cli/       21         69        131         161   ← 界面层吞掉增量的一半
```

（命令：对每个时点 `git log --until=<d> -1` 取 commit 后 `git ls-tree -r …|wc -l`。）

也就是说：**膨胀不是引擎变复杂，而是界面长进了引擎的胸腔里。**
这直接解释了第 5 节为什么 app/desktop 包被提前剖出来。

## 2. 修不动的 TUI

### 2.1 技术栈考古：三易其主

V1 TUI 的渲染基座换了三茬（全部实测）：

```
2025-03~05   Go + bubbletea（termai 原型）
2025-05-18   TS 重写期引入 ink（React for terminals）
2025-05-29   "Remove React/Ink dependencies..."（commit 实测）
             → 自研轻量渲染 + 后续 log-update（oclite 路线）
2025-10-31   "OpenTUI is here" (#2685) ── @opentui/solid + solid-js 进 package.json
             （实测：该 commit 前后 diff，opentui/solid-js 是唯一新增渲染依赖）
```

`git log --reverse | grep -cE "wip.*tui"` = **77** 条 wip:tui——2025-05-28 前后整周都在
"wip: refactoring tui"。基座三换本身就说明：没有一层是"选对了"的。

### 2.2 复发模式：修了又坏（git 实测）

V1 期（2025-07~2026-03）含 "fix" 的提交里，tui 相关 **159 条**；其中：

```
scroll 类 fix 按月分布（grep -i fix | grep -i scroll | 按月 uniq -c）：
2025-07 ████ 7        2025-12 ███████████████ 15
2025-08 █ 3           2026-01 ██████████████████████████ 26  ← 峰值
2025-11 ████████ 8    2026-02 █████████████ 13
(09/10 缺测)          2026-03 ██████ 6

render 类 fix 按月分布：7月8 → 8月3 → 11月4 → 12月7 → 1月6 → 2月8 → 3月5
```

同一类 bug 连续 9 个月按月复发，这就是"修不动"的定量画像。抽样原文（实测）：

```
2025-11-05  fix(tui): restructure Sidebar component to be scrollable (#3946)
2025-11-07  upgrade opentui to fix disappearing content (again) and sticky scroll   ← "(again)"
2025-11-11  upgrade opentui to 0.1.41 … synced rendering to fix cursor flickering…
2025-12-01  bump opentui to v0.1.54, removing the cc flicker feature…
2025-12-30  upgrade opentui to v0.1.66, fixing split diff alignment
2025-12-30  fix: revert opentui to 0.1.63 to fix streaming jitter regression        ← 升级即回滚
2026-01-11  fix(tui): slash command autocomplete highlighted row jumping (#7815)
2026-01-15  upgrade opentui to v0.1.73, fixing CJK word wrapping and thai text…
2026-02-11  fix(tui): prevent home wordmark corruption in height-constrained…(#13069)
2026-02-17  fix(tui): style scrollbox for permission and sidebar (#12752)           ← 半年后同款
```

两个铁证细节：

1. **"(again)"**：2025-11-07 的提交信息自己承认内容消失是复发 bug。
2. **升级即回滚**：2025-12-30 同一天先升 0.1.66 又回滚 0.1.63——渲染层版本像走钢丝。

OpenTUI 引入后 5 个月（2025-11~2026-03）有 **62 次 opentui 相关提交，其中 19 次带 fix**。
渲染器本身成了主要 bug 来源：flicker、jumping、disappearing、jitter、CJK 换行、
kitty keyboard、tmux 回归……全部是"终端图形学"级难题（分析：这些 bug 的根因在
渲染模型与并发状态更新不匹配，而非某个具体 patch 的疏漏，所以逐条修永远修不完）。

性能侧的老大难是 #811（Text rendering VERY slow + CPU high，2025-07-09 提出，💬84），
拖了 **4 个多月**到 2025-11-20 才 close——恰好 close 在 OpenTUI 到来前后（分析：以
换渲染器了结，而非修复了原实现）。

### 2.3 #11762：社区的换心提案（2026-02-02）

"Epic: Ink-based TUI Rewrite for oclite"——注意语境（issue 原文实测）：
它针对的是 **oclite（log-update 轻量 UI，17 文件约 2500 行）**，而非主 TUI；
痛点是 `pausedForProse` hack、乱序事件缓冲、"log-update 根本没为这个复杂度设计"。
这印证了连最轻的界面层都到了基座极限。有意思的是方向之争：社区想**退回 Ink**
（React reconciler、`<Static>` 批量状态更新），而官方最终路线是**把 OpenTUI 做成
一等公民**（2026-01-28 已有 `feat(app): use opentui markdown component behind
experimental flag`，2026-06-03 tui 独立包 + OpenTUI 定型，见下卷）。
（分析：无论哪条路，共识都是"修补已到尽头，必须换渲染架构"。）

## 3. 体内试验：局部 v2 的挣扎

V2 不是某天拍板从零起的——在 V1 体内有过两轮"器官移植试验"：

```
2025-07-07  第一代 "v2 message format"（AI SDK v5 升级，PR #743）→ 命名层试验
2025-12-07  "sdk v2"（第二代 v2 概念，实测 commit 主题即 "sdk v2"）
2025-12-10  "global bus"（实测仅 4 行 diff，bus/index.ts 加 GlobalBus 出口）
2026-04-12  src/v2/ 命名空间出现（"ignore: exploration" 系列，+v2/session.ts 71 行）
2026-04-16  "refactor: unwrap SessionV2 namespace + self-reexport (#22978)"
```

`global bus` 那笔 4 行提交的意味（分析）：在单例满天飞的 V1 里先拉一根全局总线，
让跨模块事件不必互相 import——这是给 V2 的"神经系统"打前站，但 4 行的体量说明
当时只敢做最小切口。

`SessionV2` 命名空间的挣扎最典型：2026-04-12 Dax 的 "ignore: exploration" 系列
（v2/session.ts、v2/message.ts）把 V2 会话原型直接塞进 `packages/opencode/src/v2/`
这个 V1 体内的"隔离舱"；4 天后（04-16）Kit 的 unwrap 提交把 `namespace SessionV2`
拆掉换成自导出——**原型长到第 4 天就必须做一次内部重整，否则连编译边界都不干净**。
（分析：`v2/` 目录 + "ignore:" 前缀 = 在巨石里划无菌手术区，"先在旧身体里养新器官，
成活后再整体移植"——2026-04-15/25 server/core 包诞生就是这个移植。）

命名本身也是证据：`SessionV2`、`sdk v2`、`v2 message format`——三次不同的东西
都被叫过 "v2"，说明 V2 是一个被反复逼近才逐渐成形的概念（分析级判断）。

## 4. Effect 渐进路线与安全网

Effect 化是本时期最重的外科手术，官方走的是"先织安全网再动刀"：

```
2026-03-12  branch effect-auth-foundation（实测 origin/effect-auth-foundation 末笔）
            └─ 同日 "refactor(auth): effectify AuthService (#17212)" —— 第一个器官
2026-03-12  commit 11e2c853 "chore: keep effect migration plan local"
            └─ 从 PR 里删掉 399 行的 EFFECT_MIGRATION_PLAN.md（git show 可完整复原！）
2026-03-18  commit f3cf519d "feat(bus): migrate Bus to Effect service with PubSub internals"
2026-03-19  "effectify Worktree service" / "effectify Pty service" / 四个 kit/effectify-* 分支
2026-04 下旬 httpapi parity 契约网铺开（该月 99 条 httpapi 提交，11 条带 parity）
```

**399 行迁移计划书**（`git show 11e2c853^:packages/opencode/EFFECT_MIGRATION_PLAN.md`
可全文复原）是本时期最重要的方法论文物。原文节选（实测）：

> Move `packages/opencode` toward Effect **one domain at a time**… Treat CLI, server,
> and jobs as **runtime boundaries**… Begin by extracting an Effect service behind the
> existing module boundary. Keep old call sites working by adding a **thin promise
> facade** only where needed… Use a shared app runtime **only to help mixed code
> coexist** while we migrate.

翻译成手术术语：一次只换一个器官（domain）、在边界上保留 promise 假体（facade）、
共享 runtime 只是过渡脚手架不作为最终架构。计划书里甚至点名了
`account/service.ts`（Effect 原生）与 `account/index.ts`（promise 门面）的双层结构。

Bus 迁移（03-18）是第一个中枢器官：`Bus.Service` 以 Effect PubSub 为内核，
legacy 适配器包在外面——与计划书"facade-first"完全一致。

4 月的 **httpapi parity 契约网**是"安全网"的极致：新 Effect HttpApi 后端的每一条
路由输出都要和旧 Hono 实现做逐路由比对（实测提交串：`preserve mcp oauth error
parity` / `enforce instance route parity` / `cover session json parity` /
`finish sdk openapi parity`，直到 2026-05-03 `Server.openapi() … parity-checked
against Hono output`）。**换发动机前先把旧发动机的输出录下来当验收标准。**

## 5. 界面先行：app/desktop 早分离

引擎层的分包（server/core）要到 2026-04，但界面层早了半年就动了刀（实测）：

```
2025-09-15  "feat: add desktop/web app package (#2606)" —— packages/app 诞生
            （技术栈实测：solid-js + vite，首笔即带 160 行主题插件与全套 SVG 图标资产）
2025-10-03  "chore: app -> desktop"
2025-12-22  tauri → desktop、desktop → app 两次改名（包名摇摆的痕迹）
```

为什么界面先拆？（分析级，证据为旁证）：

1. **拉力来自外部**：桌面/Web 是用户直接伸手要的产品形态（appendix §6 榜单里
   VS Code extension #11176 仍 open、桌面相关诉求高热），交付压力大、迭代节奏快，
   不拆包就被巨石的发版周期拖死。
2. **推力来自内部**：第 1 节实测 cli/ 目录吞掉增量一半文件——TUI(.tsx) 与引擎
   同包编译、同包发版，是膨胀的最大单一来源。把 Web 界面外移 = 给巨石减压。
3. **架构上恰好可切**：界面天然吃 server 的 HTTP API（sdk 包 2025-07-22 已就位），
  接口边界清晰；而引擎内部（session/provider/bus）彼此纠缠，切不动。
   所以后者只能等 Effect 化把经络理顺再拆（server 04-15、core 04-25）。

（注意：拆出去的是 **Web/桌面** 界面；终端 TUI 此刻仍嵌在 cli/cmd/tui 的 124 个
文件里动弹不得——要等 2026-06-03 `refactor(tui): extract standalone package`。

## 6. 走到尽头的修补

痛点定量化——issue 月度 created（search API `is:issue created:<月>` 抽样实测）：

```
2025-07    621
2025-08    513
2025-10    533
2025-11    793
2026-01   3094  ← 半年内 ×5（与提交峰值 2083 同月，用户涌入撞上巨石维护极限）
2026-03   2510
```

同一窗口的 fix 主题词频（appendix §4 实测）：tui 110 / session 63 / provider 27 /
scroll 26 / render 20 / windows 25 / lsp 28——**TUI 一个词超过 session 与 provider
之和**。Windows 类 fix 实测另有 74 条（含 win32 关键词），#631 老兵还在打。
LSP 28 条则说明引擎侧同样在漏（tree-sitter 双实现、诊断时序都是雷区）。

修补的边际收益清清楚楚地衰减：scroll fix 在 2026-01 达到单月 26 条的峰值——
**一个月修 26 次滚动，等于承认滚动根本没修好。**

## 时期小结

V1 膨胀期的死锁结构（分析）：

```
        单包巨石（437 文件 / 43 目录 / 一份 index.ts）
              │ 任何模块改动都可能波及全部
              ▼
   修 A 坏 B（scroll 26/月、render 跨月复发、opentui 升级即回滚）
              │ 只能继续打补丁
              ▼
   无法换基座（ink→自研→OpenTUI 三换仍不彻底；Effect 想进进不来）
              │ 直到三件事同时就位
              ▼
   ① 方法论：EFFECT_MIGRATION_PLAN（domain-by-domain + facade）
   ② 安全网：httpapi parity 契约测试（旧实现的输出成为验收标准）
   ③ 隔离舱：src/v2/ 体内试验证明新器官能成活（SessionV2 04-16 成形）
              │
              ▼
   2026-04-15/25 server/core 诞生 → V2 动工（下一卷）
```

触发 V2 的临界条件，一句话（分析）：**当"修 TUI"本身成为主要工作量（fix 词频
tui 110 断层第一）、而修补的复发周期以月为单位且不收敛时，渐进修补的期望收益
归零——剩下的问题全部是架构性的，只能换身体。** V1 用九个月从 38 文件长到
437 文件，证明了"加法"这条路能跑多远；V2 的问题则是怎么在飞行的飞机上
把机身换掉——Effect 迁移计划、parity 契约网、v2 隔离舱，就是官方给出的三个答案。

---

### 附：本文关键断言复现命令

```bash
cd /home/donghe/github/opencode
PEAK=$(git log --format="%H" --until=2026-04-01 -1)
# 峰值解剖
git ls-tree -r --name-only $PEAK -- packages/opencode/src | wc -l        # 437
git ls-tree --name-only $PEAK -- packages/opencode/src/ | wc -l          # 43
git ls-tree -r --name-only $PEAK -- packages/opencode/src/cli | wc -l    # 161
git ls-tree -r --name-only $PEAK -- packages/opencode/src/cli/cmd/tui | wc -l  # 124
# TUI 基座三换
git log --format="%ad %s" --date=short | grep -iE "wip.*tui" | wc -l     # 77
git log --format="%ad %s" --date=short -S 'opentui' --reverse -- packages/opencode/package.json | head -1
# 复发
git log --format="%ad %s" --date=short --since=2025-07-01 --until=2026-04-01 \
  | grep -i fix | grep -i scroll | awk '{print substr($1,1,7)}' | sort | uniq -c
# Effect 路线
git show 11e2c853^:packages/opencode/EFFECT_MIGRATION_PLAN.md | head -40
git log --all --format="%ad %s" --date=short --grep="parity" -i --since=2026-04-01 --until=2026-05-15
# v2 隔离舱
git log --format="%ad %s" --date=short --grep="SessionV2" --all
```
