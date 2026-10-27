# TS 重写与开源首秀：雏形期的三家牌桌（2025-05~07）

> 基线：git + 建库初期 issue 全量（appendix-data.md §5）| 考古实证 + 动机分析标注
> 本文 git 断言全部可在本地仓库 `anomalyco/opencode` 复现；issue 数据来自 REST API 实测（2026-10-22 采集）。

## 0. 时期速览

```
2025-05-14        开源首秀（首 issue #2 出现，issue 洪峰自此起）
2025-05-30        go.mod 删除，TS 重写完成（commit f3da73553c "sync"）
2025-06-02        "add permission system"（786db364d2，权限系统第一行代码）
2025-06-02~07     "wip: refactoring tui" ×72 次——Go TUI 大重构
2025-06-05        "integrate with models.dev"（db2bb32bcf，模型目录外置）
2025-06-12        "subagent" / "opencode serve" / bedrock loader
2025-06-14        首秀 issue 大清仓（大量早期 issue 同日关闭）
2025-06-19        "remove unused permission timeout"——ask() 被短路
2025-07-09        "modes concept"（模式体系诞生）
2025-07-19        "shitty hack for terrible charm bubbletea performance"
2025-07-22        packages/sdk 诞生（"chore: generate sdk into packages/sdk"）
2025-07-31        "wip: permissions" ×5 连发——ask 白名单定型
2025-08-02        packages/plugin 诞生（"wip: plugins"，时期尾声）

月提交量：05 月 301 → 06 月 654 → 07 月 650（重写冲刺 + 首秀后高热）
```

这七周是 opencode 历史上密度最高的"三线作战"：

```
线一：代码战线——Go 单体重写为 TS monorepo（内核先行，Go TUI 暂留）
线二：社区战线——开源首秀的 issue 洪峰与外部贡献者涌入
线三：架构战线——三大痛点（权限 / 渲染 / 模型生态）在同期能动地演化

三条线互相喂养：issue 洪峰定义痛点 → 痛点决定重写后的架构投入 →
架构选择又成为下一时期的债务或资产。
```

## 1. 重写后的形态

### 1.1 2025-05-30：一声不响的".sync."

重写完成的 commit 题为 `sync`（f3da73553c），正文零解释——但 diff 说明了全部：

```
$ git show --stat f3da73553c   # 178 files, +765 / -3382
  go.mod / go.sum                    ← 删除（Go 内核退役）
  app/packages/function  => packages/function
  js/                    => packages/opencode     ← TS 内核搬家 + 升格
  app/packages/web       => packages/web          ← Astro 文档/share 站
  pkg/client/gen/*.go    ← 删除（Go 代码生成产物 -2851 行）
  packages/opencode/script/release.ts             ← 新增发布脚本
  packages/opencode/bin/opencode.mjs              ← 新增 bin 入口
```

此前仓库是 `app/`（SST 部署壳）+ `js/`（TS 客户端）+ 根部 Go 模块的三层混居；
此后是干净的 `packages/` monorepo。**动机推断（分析级）**：SST 团队的主技术栈
本就是 TypeScript，Go 只是 termai 原型的顺手选择；重写不是换语言实验，
而是"回到主场"——Bun 运行时、zod schema、AI SDK 生态都在 TS 侧。

### 1.2 2025-06-05 前后的功能面

以 `f3e31130ba`（2025-06-05）为切片，TS 重写后的包结构：

```
packages/
├── opencode/          # TS 内核
│   └── src/
│       ├── app/ bus/ config/ global/ id/ util/   # 骨架
│       ├── session/ share/ storage/ server/      # 会话 + 分享页 + 服务化
│       ├── provider/ lsp/ mcp/ ripgrep/          # 三方集成
│       ├── permission/                           # 06-02 刚诞生（见 §2）
│       ├── tool/    bash edit glob grep ls lsp-diagnostics
│       │             lsp-hover multiedit patch read todo
│       │             todoread todowrite webfetch write
│       │             （每个工具 .ts 实现 + .txt 提示词成对出现）
│       └── bun/     # Bun 运行时胶水
├── tui/              # Go TUI（暂留！bubbletea/glamour/lipgloss）
├── web/              # Astro：文档 + share 页
└── function/         # SST 云函数（share 短链等）
```

注意 **TUI 仍是 Go**——重写分两步走：内核先行，TUI 靠 Go HTTP 客户端
（`pkg/client/gen/openapi.json` 被删了又重建）与 TS server 通信。这个
"TS 内核 + Go 终端"的二元结构贯穿整个 V1 时期，是理解渲染痛点（§3）的前提。

06-04~06-10 一周的提交主题勾勒出冲刺期的日常：share page 系列（og image、
SSR、lsp diag 上页、markdown 渲染）、`wip: refactoring tui` 连发、
`autodownload lsp`、`integrate with models.dev`、`claude oauth support`、
`feat: opus 4`。重写不是从零开始——首秀攒下的需求清单在两周内被快速灌进新骨架。

### 1.3 时期尾声：sdk 与 plugin

```
2025-07-22  chore: generate sdk into packages/sdk   ← OpenAPI 代码生成
2025-08-02  wip: plugins                            ← 插件机制起手
```

sdk 包由 server 的 OpenAPI schema 自动生成——"server 即 API"的形态在
重写时已埋好（server/ 目录自 05-30 就在），07 月只是把它商品化。plugin
包则承接了 6 月底社区 PR `Formatters as plugins`（#487）的思路：**把
配置面开放出去**。这两包是下一时期（生态扩张）的种子。

## 2. 牌桌一：权限模型之争

### 2.1 争议的起点

首秀当天就有人把问题挑明：

- `#9 full-auto / permissions（💬6）`——能不能全自动？敢不敢全自动？
- `#31 Denying tool continues generation`——拒绝之后模型还在跑，拒绝算什么？
- `#239 Implement permissions in TUI（💬67）`——TUI 里根本没有权限 UI

`#239` 以 67 条讨论成为建库初期讨论量第三的 issue：**权限不是功能请求，
是安全心智之争**——CLI 编码代理每天要跑几百条 bash，放权与失控只有一线之隔。

### 2.2 三段式早期演进（git 实证）

**第一段：06-02，权限系统诞生。**
`786db364d2 "add permission system"` 仅 135 行：`Permission.ask()` 发总线
事件、挂 pending Promise、等 `respond("once"|"always"|"reject")`。语义
干净（once/always/reject 三态 + 会话内记忆），但有个致命细节——

**第二段：06-19，ask 被短路。**
`d869df4fee "remove unused permission timeout"` 名为删超时，实际在
`ask()` 开头加了一行 `return`：**权限询问被整体禁用，进入全自动时代**。
（实测：`git show d869df4fee` 可见 `+ return` 出现在函数首行。）
动机推断（分析级）：TUI 侧权限 UI 未就绪，ask 挂起会卡死会话，于是先砍。
这是典型的"安全让位于可用性"的雏形期取舍。

**第三段：07-31，一夜之间补完。**
同日五个 `wip: permissions` 连发 + `docs: permissions` ×2 + wildcard 匹配，
架构定型：

```
config.permission:
  edit: ask | allow                    # 文件编辑默认询问
  bash:  ask | allow | { "<glob>": ... }   # 按命令模式白名单

bash.ts 执行链（a2191ce6fb 实测）：
  tree-sitter 解析命令
    → cd/rm/cp/mv/chmod... 参数逐一 path.resolve
      → 越出 cwd 直接 throw（硬性红线，不问）
    → glob 模式匹配（Bun.Glob）
      → 未命中任何模式 ⇒ 默认 "ask"
```

**默认 ask + glob 白名单 + cwd 沙箱红线**——这就是后来整个 permission
系统的心智原形。appendix §7 的对照（#9/#239 → 默认 ask 白名单机制）
在这一晚完成了从讨论到代码的惊险一跃。

### 2.3 牌桌上的筹码

`#31`（deny 后继续生成）暴露的是更深的语义问题：拒绝不是异常退出，
而是要作为工具结果回喂模型。第一版用 `RejectedError` reject Promise，
直到 V1 中期才演化为"deny 也是一次 tool result"的交互语义（分析级推断，
基于后续 Effect interruption 设计）。雏形期的答案是粗糙的，但三态
权限 + 白名单的骨架立住了。

## 3. 牌桌二：渲染性能先天不足

### 3.1 V1 TUI 用什么渲染（实证）

```
$ git show f3e31130ba:packages/tui/go.mod
  github.com/charmbracelet/bubbletea v1.3.4    ← Elm 架构 TUI 框架
  github.com/charmbracelet/bubbles v0.21.0     ← 现成组件
  github.com/charmbracelet/glamour v0.9.1      ← 终端 markdown 渲染
  github.com/charmbracelet/lipgloss v1.1.0     ← 样式布局
  github.com/alecthomas/chroma/v2              ← 语法高亮
  github.com/lrstanley/bubblezone              ← 鼠标区域
```

纯 Charm 全家桶：bubbletea 的 Elm 循环（Model/Update/View 全量重绘）+
glamour 的 markdown → ANSI。`wip: refactoring tui` 在 05-06~07 月出现
**72 次**——重写完成后，TUI 是持续改动最猛的目录。

### 3.2 痛根在哪

- `#811 Text rendering VERY slow + CPU high（💬84）`——长会话下每次
  按键触发全量 View 重算，glamour 对整段 markdown 重新排版渲染；
- `#265 / #220`——滚动与 diff 渲染卡顿、闪烁；
- 流式输出是最坏场景：token 逐个到达，每个 token 都可能引发
  chat 组件全量重绘 + chroma 全文高亮。

### 3.3 07-19：绝望的一天（git 实证）

```
$ git log --since=2025-07-16 --until=2025-07-20 --format="%ad %s" --date=short
  07-19  shitty hack for terrible charm bubbletea performance   ← 4699739814
  07-19  perf: more performance improvements
  07-19  perf: scroll to bottom in thread
  07-18  message rendering performance improvements
  07-18  fix(tui): ensure viewport scrolls to bottom (#1110)
  07-16  fix(tui): mouse scroll ansi parsing and perf
```

`4699739814` 这个 commit 同时做了两件事（实测 diff）：引入
`bubbletea/v2 v2.0.0-beta.4` + `bubbles/v2` 尝鲜，**并且**手写了一个
**769 行的 `internal/viewport/viewport.go`**（附带 141 行自定义语法
高亮 viewport）——v1 与 v2 两套 bubbles 并存于 go.mod，等于承认：
现成组件救不了，只能自绘。同日提交名里那句 "shitty hack" 是 V1 时期
对渲染架构最诚实的一次崩溃。

时期内 fix 主题词频（appendix §4 口径）：`render 20 / scroll 26`，
且整个 V1 生命周期 `tui 110（含 TUI 13）` 高居榜首。**渲染是 V1 唯一
从未止血的痛点**——它的终局在很远的地方：2026-06-03 OpenTUI 自研
渲染器诞生（本系列后篇）。雏形期埋下的病根，长成了重写渲染器的理由。

## 4. 牌桌三：模型生态碎片

### 4.1 需求面的碎片

首秀两周内的 issue 就是完整的生态地图：`#11 ollama 本地模型`、
`#27 codex-mini / responses api`、`#33 openrouter 多模型损坏`、
`#469 bedrock`、`#46 OpenAI compatible provider`、`#53 groq`、
`#61 OpenRouter 上的 Claude 4`……每家 API 的鉴权、参数、工具调用
格式、缓存语义全都不同。

### 4.2 供给面的挣扎（git 密度实测）

05-30~07-31 两个月内，主题含 provider/model/ollama/openrouter/bedrock/
codex/gemini/vertex 的提交共 **80 条**，密度与 TUI 重构相当。样本：

```
06-05  integrate with models.dev          ← db2bb32bcf（转折点，见下）
06-12  bedrock loader
06-24  remove accidental openai autoloader
06-25  integrate gemini-cli strategies for edit tool   ← 回应 #266
07-??  fix openrouter caching / fix openrouter caching with anthropic
07-??  better amazon bedrock caching with anthropic models
07-??  fix(#467): more granular bedrock modelID based on aws region (#482)
07-??  fix loading api key from env for openai compatible providers
```

### 4.3 转折点：models.dev

`db2bb32bcf "integrate with models.dev"`（06-05）删掉了 77 行硬编码的
`provider/database.ts`（当时手抄了 claude-sonnet-4 的价格/上下文窗口），
改为 `models.ts` 从 sst 自家的 **models.dev 目录站**动态拉取模型元数据。
**动机分析（分析级）**：模型目录是个无休止的追新游戏，与其在代码里
追，不如把元数据外置成独立产品（models.dev 本身也是 sst 系作品），
客户端只管轮询（随后即有 `poll for models.dev changes`）。

这手棋和 provider 层建立在 AI SDK 之上（`from "ai"` import Provider SDK）
一起，构成"**目录外置 + 统一抽象**"的双层答案——appendix §7 中
`#11/#33/#469 → models.dev + 统一抽象` 的对照在此落地。至于彻底的
`@opencode/ai` 自研协议层，则是 V1 后期的事；雏形期的贡献是**认清了
问题的形状**：模型碎片不是要消灭的 bug，而是要常态吸纳的目录。

## 5. 首秀爆发与社区共建

### 5.1 洪峰与清仓（REST API 实测）

仓库 04-30 建立，但首个 issue `#2 Roadmap & Existing Issues` 出现于
**05-14**——开源首秀日。首周（~05-21）25 个 issue **至今 100% closed**，
其中 8 个在创建后 7 天内关闭；到 06-14 出现一次集中清仓（`#2 #5 #7 #9
#10 #11 #17 #20 #31 #33 #41 …` 大量 closed_at = 2025-06-14，多由
TS 重写落地后的批量 triage 完成）。建库初期 7 周 723 issues / 277 PRs
（appendix §1），全部早期 issue 状态为 closed——**没有一个是烂尾**。

### 5.2 外部贡献者的扛旗时刻

05-05~07-31 期间共 **123 位独立作者**提交。社区角色肉眼可见地升级：

```
5 月（Go 时代）：补丁期
  Ed Zynda   —— 一人七连发：非交互模式 #18、可配 shell #23、
                工具面板 #24、消息历史 #30、管道输入 #51 …
  phantomreactor —— ctrl+v 贴图 #26、Codex mini #34
  mineo      —— VertexAI provider #153

6 月（TS 重写后）：功能期
  Martin Palma        —— GitHub Copilot OAuth 认证流 #305（完整 feature）
  Gal Schlezinger     —— 优化 edit 工具渲染 #463（踩进 §3 的痛点）
  Polo123456789       —— Formatters as plugins #487（→ 08-02 plugin 包）
  Timo Clasen         —— MCP server 开关 #513/#543
  Juhani Pelli        —— 修无限 summarize 死循环 #399

7 月：主力期
  带 PR 号的合入 commit：6 月 64 个 → 7 月 180 个（近 3 倍）
  样本：Windows 兼容 zip 替换 #662、bedrock 区域粒度 #482、
        Gemini 工具 schema 消毒 #1132、Azure responses API #1428
```

**分水岭在 6 月中**：此前外部 PR 多为边角修（typo/主题/键位），此后
provider 适配、Windows 兼容、缓存优化这类核心模块开始由社区扛——
维护者从"写全部代码"转为"审大部分代码"。sdk 包（07-22）诞生后，
"用 opencode 开发 opencode 工具链"的正循环进一步加速（分析级）。

### 5.3 首秀为什么能接住

22 万 star 的起点（当前 219,857，API 实测）不是营销奇迹而是吞吐奇迹：
月提交 301 → 654 的两倍跳变，恰好覆盖 issue 洪峰窗口；share page
（会话分享页）在首秀后两周内火速上线，本身就是最有效的传播载体——
每次分享都是一张产品截图。**接住洪峰的方式决定了社区留下来的样子**：
批量关闭 + 快速响应 + PR 敞开合，把围观者转化成了 §5.2 的贡献者。

## 时期小结

```
带走的三张牌桌答案（全部 git 实证）：
  权限   → 06-02 诞生 → 06-19 短路 → 07-31 定型（默认 ask + glob 白名单）
  渲染   → Charm 全家桶先天不足 → 07-19 "shitty hack" 自绘 viewport
           （病根未除，喂给 2026-06 的 OpenTUI）
  模型   → 80 条适配提交 → 06-05 models.dev 目录外置 + AI SDK 统一抽象

带走的结构资产：
  packages/ monorepo（05-30）→ sdk（07-22）→ plugin（08-02）
  subagent（06-12）→ modes concept（07-09）→ 后续 agent/ 体系的原点

带走的双重遗产：
  ✓ 正面：吞吐文化——issue 100% 清仓、PR 敞开合、123 位贡献者入场
  ✗ 负面：Go TUI 二元结构——渲染债贯穿整个 V1（tui 110 次 fix 居首）

雏形期的本质：重写不是终点，而是把三家牌桌上的争论（能不能自动、
能不能不卡、能不能接所有模型）翻译成了可演进的代码骨架。
牌局未终——每一张都在 2026 年的 V2 里见了分晓。
```

---

*本文 git 断言复现命令速查：*
`git show --stat f3da73553c`（重写）· `git show 786db364d2` / `d869df4fee`
/ `a2191ce6fb`（权限三段）· `git show f3e31130ba:packages/tui/go.mod`（渲染栈）
· `git show 4699739814 --stat`（自绘 viewport 769 行）· `git show --stat
db2bb32bcf`（models.dev）· `git log --format="%s" --since=2025-05-01
--until=2025-08-01 | grep -icE "provider|model|..."`（=80）· issue 数据：
REST API `/repositories/975734319/issues?sort=created&direction=asc` 前 3 页
