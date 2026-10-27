# V2 重建：三次豪赌与引擎热替换（2026-03~10）

> 基线：git 全史 + [appendix-data.md](appendix-data.md) §3 | 实证 + 决策动机分析标注
> 代码基线：dev 分支（2026-10-22 采集）

```
2026-03-12 ──── 04-15 ──── 05-30 ──── 06-03 ──── 06-24/25 ──── 07-20 ──── 08-11/12 ──── 10-07 →
    │             │           │           │            │            │            │            │
 effect 迁移    server     core/event   runner+      schema/      V1 主入口    claims +     plugin/
 计划落盘       脚手架     事件溯源     OpenTUI      protocol/    删除         inbox        runtime
 (V1 内预演)    +666行     起点打开     进 V2        client       (V1 退役)    (最后定型)   定型
                                                ┌───── 3 个月双线作战 ─────┐
                                                04-15 ───────────────── 07-20
```

第四篇，收官。前三篇讲了 Go 原型八周弃船、TS 重写、V1 从 117 文件膨胀到 437 文件的修罗场。这一篇讲结局：他们怎么在**V1 还在每天漏血的同时**，把整台引擎换了。

---

## 0. 时期速览

```
月份     提交量   时期主线（实证：附录 §2/§3）
2026-03    963   effect 迁移计划在 V1 包内预演（effectify 流水开闸）
2026-04   1328   server(04-15)/core(04-25) 诞生；同期 V1 httpapi parity 测试网铺开
2026-05   1551   Effect-native 事件系统(05-14)；Effect beta 高频追踪
2026-06   1127   runner/skill/命令注册表/OpenTUI V2 化；schema/protocol/client 两日抽离
2026-07   1601   durable 化铺开；restart 恢复；mini 第二界面；07-20 V1 主入口删除
2026-08   1984   claims(08-11)/inbox(08-12)；V1 善后迁移收尾
2026-09   1486   Windows 签名终章(#631)等平台收尾
2026-10    663   plugin/runtime 定型（月中仍在高位）
```

三个数字先摆出来，后面反复引用：

- **43** 个 `effectify` 前缀提交（2026-03~05，V1 包内逐服务改造）；
- **551** 个 fix 提交落在 `packages/opencode`（2026-04-15 ~ 07-20，双线作战期）；
- **-328,689 行 / 700 文件**——2026-07-20 `removed packages/opencode` 一个 commit 删掉的全部 V1（实证：`git show 44b6938b2a --stat`）。

---

## 1. 豪赌一：Effect 全家桶

### 1.1 不是一时兴起，是三个月的"包内预演"

V2 动工（04-15 server 诞生）**之前一个月**，git 里已经出现了完整的排练痕迹。effectify 改造的服务谱系（43 个提交，按时间排布）：

```
2026-03-12  keep effect migration plan local        ← 迁移计划文档落进仓库
2026-03-12  refactor(provider): effectify ProviderAuthService
2026-03-12  refactor(auth): effectify AuthService
2026-03-13  feat(schema): scaffold effect-to-zod bridge
2026-03-16  effectify Skill/File/Format/FileTime/Vcs 五个服务（一天五个）
2026-03-17  effectify Truncate/Snapshot，删除自研 Scheduler
2026-03-18  refactor(effect): unify service namespaces
2026-03-18  migrate Bus to Effect service           ← 总线先行
2026-03-25  effectify Bus service: Effect PubSub + InstanceState
2026-03-26  effectify Config/LSP
2026-03-27  effectify Skill/Plugin 内部 + tool-registry 让出 Config/Plugin 服务
2026-03-27~31 effectify Session/SessionRevert/SessionSummary/SessionCompaction/
            SessionPrompt/Provider/Storage/…
```

改造的顺序本身有信息量：**先打地基（AuthService → Bus → Config/LSP），后搬主楼（Session 全家）**。Session 被拆成 revert/summary/compaction/prompt 四次分别 effectify——不是一口气吞下，而是承认 Session 太大、必须分片施工[分析]。这就是后来 `packages/core` 的雏形——但当时它们全住在 V1 的 `packages/opencode/src` 里。V2 不是"另起炉灶写新代码"，而是**把 V1 的服务一个一个泡进 Effect，泡熟了的才搬出去**。

server 诞生 commit 本身就是铁证：`feat(core): bootstrap packages/server and document extraction plan (#22492)`（2026-04-15）带了一份 **666 行的 `specs/effect/server-package.md`**，逐条写明 extraction 规则、包目标布局（core/server/cli/sdk/plugin）、甚至引用了姊妹仓库 opentunnel 的 `definition/`+`api/` 目录范式作为结构参照（实证：`git show 6706358a6e`）。这不是试水，是带着完整施工图的开工。

### 1.2 追着 beta 跑的四个月

Effect 4.0 全程 beta，升级流水（实证，`grep -E "bump|update" effect`）：

```
beta.42 (03-27) → beta.46 context APIs (04-10) → beta (04-27, 05-02)
→ beta.65 (05-11) → beta (05-20) → beta 74 (06-02) → beta 83 (06-21)
→ beta.98 (07-17) → beta.107 (08-17)
```

八次主动追版，中途还有 `fix: restore recent test regressions and upgrade effect beta`（03-19）——beta 破坏性变更打翻过测试。**[分析]** 追 beta 而非锁旧版，说明团队赌的是 Effect 4 的最终形态（Context/Rpc/HttpApi 一体化），宁可反复适配也不在 3.x 上做无用功。代价是真实的：43 个 effectify 提交横跨两个月（03-12~05-30），其中 Session 一个服务拆成 revert/summary/compaction/prompt 四次分别 effectify——**服务粒度的改造半径本身就是 Effect 学习曲线的度量**[分析]。

beta 追踪停止在 beta.107（08-17）——此后再无 bump 提交，恰与 V2 核心机制全部定型（08-12 inbox）的时间吻合。**[分析]** 追版是手段不是习惯：契约层和事件模型冻结后，Effect 的后续变更不再传导进核心代码，追版的必要性自然消失。

### 1.3 赌赢的判据

到 2026-05 之后，`effectify` 这个词从 commit log 里消失了——不是放弃了，是**没有剩下的服务可改造了**。新代码（core 的 runner、event、database）生来就是 Effect-first；`effect-drizzle-sqlite`、`effect-sqlite-node` 两个自建桥接包（现存于 packages/ 目录，实证）替代了裸 await 式数据库访问。第 3 篇讲过的"Effect 化两难"（V1 时期半 Effect 半 Promise 的双轨痛苦），最终以**全量下注**告终。

---

## 2. 豪赌二：自研 OpenTUI

### 2.1 #11762 的岔路口

时间线（实证）：

```
2025-05-29  Remove React/Ink dependencies     ← V1 出生 4 周后就抛弃了 Ink
2025-10-31  "OpenTUI is here" (#2685)          ← V1 时期已切换到自研 OpenTUI 渲染
2026-02-02  issue #11762: 社区提出 Ink-based TUI Rewrite
2026-05-07  introduce opentui keymap as sole key/cmd engine
2026-06-03  tui 包 + OpenTUI 深度定制进入 V2（diff hunk 导航等）
2026-06-07  refactor(tui): extract standalone package (#31193)
```

关键事实：**社区用 6.7k 讨论量的渲染慢问题（#811，第 2 篇）要求换 Ink 时，Ink 早在半年前就被这个团队扔掉过一次**。#11762 提出用 Ink 重写 TUI，团队的回答不是采用，而是把自研的 OpenTUI 从 V1 的依赖升格为 V2 的战略资产。**[分析]** 动机推断：Ink = React reconciler over terminal，而 opencode 的桌面端（app 包，Solid 风格数据流）与 Web 端已经自成体系；OpenTUI 的渲染管线对齐 Solid 响应式模型，意味着 **TUI/Desktop/Web/Server 四端可以共享同一套组件语义**——四端同构的收益大于 Ink 生态的现成组件。这属于"分析"级判断（无官方文档直说），但有一条实证旁证：V2 tui 包的组件文件全是 `.tsx` 却不依赖 react 包，且 `refactor(tui): replace v2 sync with data context`（06-10）显示 TUI 侧在向 core 的数据上下文模型靠拢，而非向 React 靠拢。

### 2.2 版本流水里的坚持

V1 时期 OpenTUI 就是外部依赖的滚动升级（0.1.68 → 0.1.91，2026-01~03，实证 `--grep opentui`），每次升级都捎带修复：`fixing CJK word wrapping and thai text rendering`（01-15）、`fixing capability detection and kitty keyboard`（01-09）。V2 时期升级到 0.2.x/0.3.x（05-20 → 06-08），中间还出现过 `upgrade → revert → revert the revert` 的三日反复（06-02 两次 + 06-04 恢复，实证）。

60FPS / kitty 键盘协议这些目标来自 OpenTUI 项目本身的定位（笔记 2_tech-stack/3_opentui 有走读）；对 opencode 而言，自研渲染器的直接回报是**渲染类 fix 从 V1 词频榜前列（tui 110 + render 20，附录 §4）降到 V2 时期的零星样式微调**（06 月 tui 提交里已全是 feature 而非 perf/fix 事故）。

### 2.3 tui 包的 V2 化与 mini 第二界面

V2 的 tui 不是 V1 TUI 的搬运，是一次彻底重写后的重新生长：

```
2026-05-07  opentui keymap 成为唯一按键/命令引擎（#26053）
2026-06-03  runner/skill 落地同日，tui 包开始吃 core 的 V2 服务
2026-06-05  feat(tui): add diff hunk navigation —— V2 新能力，V1 时代做不到的增量
2026-06-07  refactor(tui): extract standalone package (#31193)
2026-06-10  refactor(tui): replace v2 sync with data context —— 与 core 数据上下文对齐
2026-07-19  tui/mini 第二界面（diff-filter=A，实证）
2026-07-20  migrate diff viewer to V2 theme / session timeline / categorical theme hues
```

**[分析]** 07-20 这一天 tui 的提交密度（主题迁移、时间线、主题色相，同日多个 feature）说明 V1 删除当天 tui 侧是在"庆祝式冲刺"——引擎舱已经换完，方向盘终于属于自己的了。而 mini 第二界面（07-19）选在 V1 删除前一天诞生，时机耐人寻味：双屏形态只有在一套渲染器、一套主题系统完全自持之后才敢做——**Ink 时代不可能有 mini**，因为每加一个界面就要多养一份 React reconciler 的终端适配[分析]。

---

## 3. 豪赌三：事件溯源推倒重来

### 3.1 两表直写 → durable 事件 + 投影

V1 的存储：session/message 两张表直写，崩溃/并发/重放全靠事后修补（第 3 篇的 session 63 次 fix 是直接代价）。两表直写的结构性缺陷可以列成一张故障传导表[分析]：

```
两表直写模型                     V1 时期的对应症状（第3篇词频/issue）
──────────────────────────────────────────────────────────────
写入路径 = 业务路径               任何业务 bug 直接污染持久化数据
无重放能力                        数据坏了只能手修库（sessions lost 系列 fix）
并发写无串行化点                  worktree/orphan branch 丢 session（#16389/#16814）
中断语义落在内存                  #41/#31 中断后状态错乱
```

V2 的答案在 git 里有一个清晰的诞生序列：

```
2026-05-14  Add Effect-native core event system (#27415)   ← 事件系统落地
2026-05-30  refactor(core): move database schema ownership  ← event/ 目录成为真相源
2026-07-02  durable log reads, changes feed, watermarked snapshots
2026-07-06  durable compaction barrier / durable instruction
2026-08-13  refactor(core): remove bus replay all          ← 投影重放路径收敛定型
```

终态即笔记 6_modules/1_data-layer.md 的一句话总纲："**事件是真相源，表是投影**"——`Bus.publish` 在同一 SQLite 事务里追加 `event` 表、跑投影写读模型，崩溃后可清空投影重放重建。V1 的"写坏了就是写坏了"变成 V2 的"写坏了就重放"。

### 3.2 1135 行的诚意

`packages/core/src/database/v1-migration.bun.ts`——**1135 行，只干一件事：把 V1 用户的旧库（session/message 两表）搬进 V2 的事件模型**（实证：文件现存，另有 `.noop.ts` 给非 bun 平台空实现 + 独立测试文件）。加上 `database/migration/` 下 50 个手写迁移文件（笔记 6_modules/1_data-layer.md 有细节），V2 的存储体系从第一天就把"老用户怎么办"当成一等工程问题。

**[分析]** 22 万 star 的项目推倒存储模型，最大的风险不是技术而是用户信任——历史上无数"重写毁产品"的案例都死在数据迁移。1135 行迁移脚本 + 07 月底 desktop 端 `show migration progress`（08-14，实证）说明这是全公司级别的承诺，不是甩给用户的"export/import 自己想办法"。

### 3.3 推倒的勇气从哪来

**[分析]** V1 时期与状态相关的三大 fix 族：session 63 次、中断/重启类（#41/#31 附录 §5）、崩溃丢状态（附录 §7 专门一行）。两表直写模型下，这些问题的修复都是**在错误的真相源上打补丁**。事件溯源不是"更好的存储"，是"让这类 bug 无法再发生"的模型级消灭——这解释了为什么它值得 3 个月重写 + 1135 行迁移。

---

## 4. 双线作战的纪律（V1 善后与 parity 网）

最容易被忽略、但可能是整个 V2 最了不起的部分：**2026-04-15 ~ 07-20 这三个月，V1 一步没停**。

```
双线作战期 packages/opencode 内的提交（实证）：
  fix 类     551 个        ← V1 每天仍在漏血、仍在止血
  总提交     614 个        ← 另有 parity/重构混在其中
同期 packages/core+server 从 0 长到 283+269 commits/月（06/07 月，实证）
```

怎么做到不翻车？git 给出的答案是**契约测试网（parity net）**：

```
2026-04-27  fix(httpapi): enforce instance route parity (#24660)
2026-04-27  test(httpapi): cover full OpenAPI route inventory (#24667)   ← 全路由清单化
2026-04-27  fix(httpapi): preserve mcp oauth error parity (#24706)        ← 连错误都对齐
2026-04-27  fix(httpapi): preserve provider oauth authorize parity
2026-04-27  test(httpapi): cover session json parity (#24682)
2026-04-28  fix(httpapi): finish sdk openapi parity (#24827)
2026-04-29  Prepare Effect HttpApi backend parity (#24853)
2026-04-29  test(httpapi): clean up SDK parity tests
```

三天之内，把 V1 的 OpenAPI 路由全量做成 V2 HttpApi 的**等价性断言**——新引擎（server 包，Effect HttpApi）每实现一条路由，都必须在 parity 网上跟 V1 的行为逐字节对齐才算完成。之后整个 5 月还在持续加密这张网（`migrate httpapi tests to effect runner` 系列，05-12~13，实证）。

**[分析]** 这是经典的"strangler fig（绞杀者模式）+ 特征测试"纪律：不预测旧系统行为，而是**把旧系统的可观察行为冻结成测试**，让替换件以通过测试为完成标准。"边开边换引擎"能成立，靠的不是胆子大，是这张网兜底——07-20 删掉 700 文件时，parity 网已经跑了近三个月。

双线期 V1 侧修的都是什么？抽 04 月下旬几条看（实证，均落在 packages/opencode）：

```
2026-04-15 后的 V1 fix 抽样（与 server/core 诞生同期）：
  fix(opencode): sessions lost after git init in existing project (#16814)
  fix(opencode): lost sessions across worktrees and orphan branches (#16389)
  fix: filter empty content blocks for Bedrock provider (#14586)
  fix: non openai azure models that use completions endpoints (#17128)
```

丢 session、provider 兼容——全是第 3 篇点过名的老病灶。**[分析]** 值得注意的是团队没有因为"反正要换引擎"而放任 V1 腐烂：每一个 V1 fix 都同时是 parity 网上的一条新断言素材——**V1 的止血工作直接喂养了 V2 的验收标准**，这是双线作战能自我造血的关键一环。

---

## 5. 包拓扑诞生顺序解读

```
04-15  server   ← 先于 core 诞生！
04-25  core     ← 由 "shared 包" 更名而来（#24309，实证）
05-30  core/src/event
06-03  runner / run-coordinator / skill / tui(+OpenTUI)
06-24  schema
06-25  protocol / client
07-09  execution/restart
07-19  tui/mini
10-07  plugin/runtime
```

三处反直觉的顺序，各有一段解读：

**① server 先于 core（04-15 vs 04-25）**——不是"先定 API 形状再填引擎"的浪漫故事，实证更朴素：server 诞生时只是个空脚手架（bootstrap commit 只有 package.json + 3 行 index.ts + 一份 666 行施工图），而 core 的实体（各 Effect 服务）当时**还在 V1 包里被 effectify**。真正的顺序是：**计划先行（server-package.md）→ 服务在旧包内改造 → 改造完成的资产抽成 shared 包 → 更名 core**。"core 是搬出来的，不是写出来的"[分析]。

**② schema/protocol/client 挤在 06-24/25 两天**——

```
06-24  refactor(schema): extract shared public schemas (#33571)
06-25  refactor(protocol): extract server contracts (#33708)
06-25  feat(sdk): add HttpApi clients and embedded host (#33445)
```

两天三个契约包，这不是渐进演化的节奏，是**一次性决断**：schema（公共形状）、protocol（服务端契约）、client（消费端）三层在同一周切干净。这三个包的依赖方向决定了整个 monorepo 的骨架：

```
        schema（纯形状，无依赖）
        ↑         ↑
    protocol     client
    （服务端契约）（消费端 SDK）
        ↑         ↑
       server ── core ── tui/app
```

**[分析]** 契约层最难的部分不是写，是"划界"——哪些类型算公共、哪些算服务端私有，这个决定牵动所有包的依赖方向。选择在 06 月下旬（core 服务已基本搬完、tui 已独立）一刀切，说明他们等到了**领域边界稳定之后才划契约**，避免契约层反复 churn。对照 server 包施工图（04-15 的 666 行文档）里"postpone the runtime composition root until packages/core exists enough"的原则——契约层的抽离同样遵循"等下游成熟再划线"的节奏，而不是一开始就拍板。

**③ plugin/runtime 最晚（10-07）**——插件是纯消费者，宿主没定型它就无法定型（见 §6）。

顺带一组规模数据佐证"core 是搬出来的"：双线作战期 core+server 的月提交 06 月 283 + 07 月 269（实证 `git log -- packages/core | 按月计数`）——加上同期落在 V1 包内的 614 个，V2 的真实工程量远超任何单一包的表面数字。

---

## 6. 收尾期：最晚定型的两个机制

07-20 V1 删除后，V2 并非就此"完成"。收尾期的完整时间线：

```
07-20  removed packages/opencode（-328,689 行 / 700 文件）
07-24  fix(app): hydrate v1 session progress            ← V1 兼容长尾开始
07-24  fix(app): restore directory-scoped session status for v1 servers
07-26  refactor(sdk): remove local legacy package
07-26  bus.ts 定型 / 工具架构合并                       ← 事件总线终稿
08-11  write-ahead execution claims (#41800)
08-12  generic session inbox (#42109)
08-13  refactor(core): remove bus replay all            ← 投影重放路径收敛
08-14  feat(desktop): show migration progress           ← 迁移体验还在打磨
10-07  plugin/runtime 插件宿主定型
```

两个核心机制拖到了 8 月中旬：

**write-ahead execution claims（08-11）**：把"执行权声明"从关闭钩子里的内存挂起，改为**先写库再执行**的 write-ahead 记录。进程在任何时刻崩溃，重启后按 claim 表恢复现场。注意 commit 措辞——`replace shutdown-hook suspension`——它推翻的是 V2 早期自己的方案（shutdown-hook），等于收尾期还在推翻自己的中间设计。

**generic session inbox（08-12）**：消息先入收件箱（准入），再由执行侧认领（执行）——准入与执行分离。社区 issue #4821（unqueue messages，👍126，附录 §6）四年后由 inbox 的 cancel/delivery.changed 正式回应。

**[分析] 为什么这两个最晚？** 因为它们防御的是**最难复现的故障域**：不是"代码逻辑错"而是"进程在错误的时间点死掉"。崩溃安全的推理需要整个事件模型、恢复路径、durability 语义全部定型之后才有意义——07 月的 durable barrier（07-06）、restart 恢复（07-09）都是它们的前置。**准入/执行的分离则是并发模型的最深层**：先有"事件是真相源"（05-30），才有"什么该成为事件"（08-12）。最难的课永远最后补上，因为它的考题由前面所有章节写就。

至于 plugin/runtime（10-07）——插件宿主必须是最后一个：插件是纯粹的消费者，它寄生在宿主的全部 API 之上；宿主的 Effect 版本、服务边界、事件模型每动一次，插件 ABI 就要跟着动一次。收尾 commit `fix(cli): give plugins the host's Effect`（10-07，实证）本身就是这个问题的注脚：连"插件该用哪份 Effect 运行时"都是直到定型当天才彻底想清楚的细节。

同期善后流水（实证）：`refactor(sdk): remove local legacy package`（07-26）、desktop 的 `hydrate v1 session progress` / `restore directory-scoped session status for v1 servers`（07-24）——V1 退役后的兼容长尾又拖了近一个月。

---

## 7. 回望与全卷收官

### 7.1 V1 痛点 → V2 解法 → 代价

对照第 3 篇的痛点清单（V1 fix 词频 + 高票 issue）：

```
V1 痛点（第3篇）            V2 机制                        代价
────────────────────────────────────────────────────────────────
渲染慢 #811 / tui 110 fix   OpenTUI 自研渲染器（Solid 模型）  自研维护负担、0.1.x→0.3.x
                            + 四端同构组件语义               反复升级（含三日 revert）
单包巨石 437 文件           core/server/schema/protocol/    包间依赖方向的心智成本
                            client/tui/plugin 拓扑          + parity 网的持续维护
写不动（半 Effect 双轨）     Effect 全家桶全量下注            学习曲线（43 次服务级改造）
                                                           + 追 beta 的 8 次适配
崩溃丢状态/session 63 fix   durable 事件 + 投影 + inbox     事件模型的复杂度
                            + write-ahead claims            （重放/序列号/屏障全套）
老用户数据                   v1-migration 1135 行 +          迁移本身的长期养护
                            50 个增量迁移                    （.bun/.noop 双实现）
```

一句话：**V2 用"模型级消灭"替换了 V1 的"补丁级生存"**，而支付的货币是复杂度——只是这一次，复杂度被契约层（schema/protocol）、测试网（parity）和迁移脚本管在了笼子里。

值得对照的是月提交量的走势（附录 §2）：V2 重建期（2026-04~08）月均 1500+，**高于** V1 末期（2026-02 的 1165 / 03 的 963）。**[分析]** 重写并没有带来一段"轻装上阵"的产出低谷，反而是在双倍战场（V1 维护 + V2 铺设）上叠出了历史第二高的提交强度——推倒重写的真实成本不是写得慢，是**三个月里每件事都要做两遍**。这也是多数团队不敢重写的根本原因：不是不相信新架构，是不相信组织能撑住双线期。

### 7.2 四段史各一句话

```
【1】Go 原型（2025-03~05） 8 个 commit 试错，8 周弃船——语言选错就立刻认栽。
【2】TS 重写（2025-05~07） 8 周重写上线，出生四周就扔掉 Ink——快比完美重要。
【3】V1 膨胀（2025-07~2026-03） 437 文件修罗场，110 次 tui fix——补丁终有极限。
【4】V2 重建（2026-03~10） 三次豪赌 + parity 网双线作战——引擎热替换的教科书。
```

### 7.3 全卷收官：痛点 → 机制对偶表

四篇贯穿同一条线索：**社区的困难清单与代码库的机制清单互为镜像**。收卷总表（详版见 [appendix-data.md](appendix-data.md) §7）：

```
#9/#239  权限争议        →  permission 系统（ask 白名单 + deny 前置）
#811     渲染慢          →  OpenTUI 自研渲染器            【本篇 §2】
#41/#31  中断语义        →  Effect interruption + resume
崩溃类   丢状态          →  durable 事件 + claims + inbox  【本篇 §3/§6】
#4821    unqueue         →  inbox cancel/delivery.changed  【本篇 §6】
#3235    Skills          →  skill 注册表（06-03）          【本篇 §0 编年】
#631     Windows         →  三平台 #sqlite + 签名流水线（09-11 终章）
```

最后一组数字：2026-10 月中，V2 已定型 13 个月的项目还有 **663 commits/半月**（附录 §2）。推倒重写从来不是终点——它只是把下一个 V1 膨胀期的时钟重新拨快了一点。区别在于，这一次他们知道时钟在哪里。

---

## 附：本篇考古命令（可复现）

```bash
git log --diff-filter=A --format="%ad %h %s" --date=short -- "packages/*/package.json"
git show 6706358a6e            # server bootstrap + 666 行 extraction plan
git show 44b6938b2a --stat     # V1 删除：700 files, -328689
git log --format="%s" | grep -c effectify                      # 43
git log --format="%s" --since=2026-04-15 --until=2026-07-20 -- packages/opencode | grep -c "^fix"  # 551
git log --grep="parity" --since=2026-04-25 --until=2026-05-01  # parity 网三日铺设
wc -l packages/core/src/database/v1-migration.bun.ts           # 1135
ls packages/core/src/database/migration | wc -l                # 50
```

· 上一篇：[3_v1-growth-pains.md](3_v1-growth-pains.md) · 数据附录：[appendix-data.md](appendix-data.md) · 总览：[README.md](README.md) ·
