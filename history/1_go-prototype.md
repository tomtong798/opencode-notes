# Go 原型期：termai 的八周（2025-03~05）

> 基线：git 全史考古 + [appendix-data.md](appendix-data.md) | 可信度：考古为实证，动机为分析

## 0. 时期速览（时间线小图）

```
2025-03-21 ── 2025-05-30 ：Go 原型的完整生命周期（约 10 周，"八周"指 3-21 到 5-17 重写启动）

03-21   4b0ea68d7a  initial ────────────── termai 诞生：bubbletea TUI 骨架，还没有 agent
03-23   e7258e38ae  initial agent setup ── LLM 接入，工具链开建
03-25   904061c243  additional tools ───── 一口气 10 个工具（edit/grep/glob/ls/view/write...）
03-27   afd9ad0560  rework llm ─────────── anthropic-sdk-go + openai-go 双官方 SDK
03-末   cfdd687216  add initial lsp support（+13991 行的大 commit，LSP 全家桶）
04-05   PR #21 合并 ────────────────────── adamdottv 等外部贡献者涌入
04-16   cc07f7a186  rename to opencode ── termai → opencode
04-30   GitHub 仓库创建 ────────────────── 代码先行 40 天才建仓
05-17   a34d020bc6  sync（Dax Raad）───── TS 重写第一 commit：js/ 目录出现
05-23   83974e0c95  Share: sync ────────── SST app/ 目录出现（share page 后端）
05-29   d398001f96  Remove React/Ink ──── TS 端 TUI 路线反复：先 Ink 后弃
05-30   f3da73553c  sync ──────────────── monorepo 大重组：go.mod 从根目录消失
05-30   f50a57041f  embed go ───────────── Go TUI 嵌入式发布（未死，降级为包）
```

月度提交量（appendix-data.md §2）：`03 月 8 → 04 月 152 → 05 月 301`。3 月是纯原型（8 个 commit 搭出全部骨架）；4 月是 Go 版功能冲刺，且外部贡献者已达 18 人（`git log --format='%an' --since=2025-04-01 --until=2025-04-30 | sort -u`）；5 月的前半仍是 Go 版迭代 + 社区 PR，后半（05-17 起）被 TS 重写淹没——5 月 Dax Raad 一个人打了 102 个 commit，与 adamdottv（120 个）并列当月主力。

复现：

```bash
git log --reverse --oneline | head -30          # 前 30 commits 轨迹
git log --oneline --diff-filter=D -- go.mod     # → f3da73553c (2025-05-30) sync
git log --oneline --diff-filter=D -- cmd/termai/main.go   # → 8daa6e774a (2025-03-23) 已被重构掉
git log --oneline --since=2025-05-25 --until=2025-06-05 | wc -l   # → 233，重写冲刺密度
```

## 1. 原型考古：termai 是什么样

### 1.1 首 commit：先有终端，后有 agent

首 commit 就是一副"认真做产品"的骨架，不是玩具：

```
$ git show 4b0ea68d7a --stat          # 2025-03-21，28 files, +2229
cmd/termai/main.go                    # 入口（59 行）
internal/logging/                     # logger/writer/message 四件套
internal/pubsub/                      # broker(101 行)/events —— 事件总线，组件解耦的地基
internal/tui/
  ├─ components/                      # logs/table.go, repl/{editor,messages,threads}
  ├─ layout/                          # bento.go(361 行!), border.go, single.go
  ├─ page/                            # init.go, logs.go, repl.go —— 页面路由雏形
  └─ styles/                          # markdown.go(498 行!), styles.go, icons.go
```

注意两个反常信号：首 commit **没有任何 LLM 代码**（`internal/llm/` 两天后才出现），却有 361 行的 bento 布局和 498 行的 markdown 渲染——作者 Kujtim Hoxha 的第一优先级是 TUI 本身。`styles/markdown.go` 498 行手写样式，说明终端渲染 markdown 是他自己啃的硬骨头。`internal/pubsub/broker.go` 从第一天就在——事件驱动架构是这个项目的胎记，后来一路演进为 TS 版的 `bus/`。

### 1.2 技术选型：charmbracelet 全家桶

技术选型（`git show 4b0ea68d7a:go.mod`，全部 charmbracelet 系）：

| 用途 | 库 |
|---|---|
| TUI 框架 | `charmbracelet/bubbletea v1.3.4`（Elm 架构：Model/Update/View） |
| 组件 | `charmbracelet/bubbles v0.20.0` |
| 样式 | `charmbracelet/lipgloss v1.1.0` + `catppuccin/go`（配色） |
| Markdown | `charmbracelet/glamour v0.9.1` |

LLM 接入用的是**双官方 SDK**（`git show afd9ad0560:go.mod`）：`anthropics/anthropic-sdk-go v0.2.0-beta.2` + `openai/openai-go v0.1.0-beta.2`——注意都是 beta 版，Go 的 LLM SDK 生态当时远未成熟。没有用 langchaingo 之类的封装层，LLM 调用层是自己写的（03-23 `e7258e38ae` 的 `internal/llm/llm.go` 206 行 + `models/models.go` 196 行）。

### 1.3 工具面：一夜成型，权限同步落地

工具面在 03-25 `904061c243 additional tools` 一夜成型（`--stat` 实证 10 个工具文件）：

```
internal/llm/tools/
  ├─ agent.go(141 行)  ├─ bash.go   ├─ edit.go(420 行)  ├─ file.go
  ├─ glob.go           ├─ grep.go   ├─ ls.go            ├─ view.go
  └─ write.go          + internal/permission/（权限系统同步出现）
```

两个产品原则在此定型：

- **subagent 从第一天就存在**（`agent.go`）——不是后加的功能，是原设想的核。
- **权限对话框与工具同 commit 出现**（`components/dialog/permission.go`）——"每个工具都要过权限"从原型期就是产品原则，后来成为 opencode 的标志性交互。

03-末的 `cfdd687216 add initial lsp support` 是个 +13991 行的巨型 commit，包含 `internal/lsp/watcher/watcher.go`（633 行）等完整 LSP 实现——比大多数同类项目早半年把 LSP 做进 agent。

### 1.4 termai 时期功能面（从提交信息推断）

```
03-23  add help / colors added             基础交互
03-23  initial agent setup / working agent agent 跑通
03-25  additional tools                    工具矩阵
03-27  rework llm                          双 SDK provider 层
03-末  add initial lsp support             LSP 全家桶
03-末  add termai config                   配置文件
03-末  add sourcegraph tool                第一个"网络型"工具
04 月  diff 着色 / 动态权限对话框 / 状态栏超时 / bash 安全加固 / bedrock 支持
```

4 月起外部贡献者涌入：PR #21（anthropic 空消息）、429 重试、GPT-4.1 配置、LSP 进程清理——社区在 Go 期就已成形，这是"弃船"决策风险高的原因，也是后文的重要背景。

到 5 月中，Go 版工具面已达 17 个（`git ls-tree d0ad09d798:internal/llm/tools/`）：

```
bash.go  batch.go  edit.go  fetch.go  file.go  glob.go  grep.go  ls.go
patch.go view.go  write.go + 6 个 LSP 工具：
lsp_code_action / lsp_definition / lsp_diagnostics / lsp_doc_symbols
lsp_references / lsp_workspace_symbols
```

## 2. 重写冲刺：八周从 Go 到 TS

### 2.1 时间线修正：05-30 不是"重写完成"，而是"重写转正"

TS 代码首现于 05-17，出自 Dax Raad 之手（这是他主导重写的第一个 commit，但考古显示他 4 月就已作为贡献者出现）：

```
$ git log --reverse --oneline -- js/          # TS 代码第一次进仓库
a34d020bc6 2025-05-17 sync（Dax Raad）
  ├── js/src/app/index.ts   ├─ config/config.ts   ├─ id/id.ts
  ├─ server/server.ts       ├─ session/session.ts ├─ storage/storage.ts
  └─ util/{context,log}.ts
```

结构不是翻译 Go，而是全新分层：`server/`（HTTP server，OpenAPI 契约先行）+ `session/` + `storage/`——Go 原型里没有 server 层。此后 13 天（05-17~05-30）Dax 以每天约 8 个 commit 的速度推进重写。

### 2.2 "翻译"还是"重做"？——两条证据链说"重做"

**证据一：契约驱动的架构反转。** 06-03 `caf9fdc893 initialized` 带来 `packages/tui/pkg/client/gen/openapi.json` + `generated-client.go`（252 行生成代码）——Go TUI 变成了 TS core 的 **API 客户端**。架构从"TUI 是主体"反转为"core 是主体、TUI 是渲染端"。这个反转意味着 Go 代码无论多好都无法保留在核心路径上。

**证据二：功能面对比。** TS `packages/opencode/src/tool/`（`git ls-tree caf9fdc893:`）：

```
bash.ts  edit.ts  example.ts  fetch.ts  glob.ts  grep.ts  ls.ts
lsp-diagnostics.ts  lsp-hover.ts  patch.ts  view.ts
```

工具集基本对齐 Go 版（6 个 LSP 工具收敛为 2 个），**没有丢功能**；同时新增 Go 版没有的 `share/`、`bus/`、`server/`、`global/`。所以定性是"**重做而非翻译**"：工具逻辑是语义级移植（edit/grep/glob/ls/view/write 一一对应），架构是重设计（server/client 分层是全新的）。

### 2.3 重写冲刺的日常

05-25~06-05 的 233 个 commit 里，`wip: refactoring tui` × 5、`share page` 系列密集出现——这正是在把 Go TUI 塞进 client/server 模式。期间还有一次路线反复值得记录：**05-29 `d398001f96 Remove React/Ink dependencies`——TS 端曾短暂试过 Ink（React 渲染终端）做 TUI，随即放弃**，回归 Go bubbletea 作为终端前端。这条 commit 的说明里还带了一句 `🤖 Generated with opencode`——opencode 已经开始写 opencode，自举在重写第一周就发生了。

### 2.4 分水岭：05-30 大重组

`f3da73553c sync`（Dax Raad，178 files, +765/−3382）：

```
go.mod          → packages/tui/go.mod      （根目录 go.mod 删除 = 基线断言的由来）
cmd/            → packages/tui/cmd/
internal/       → packages/tui/internal/
js/packages/opencode → packages/opencode/  （TS core 升上第一层）
app/sst.config.ts     → sst.config.ts      （SST 提到仓库根）
```

同日 `f50a57041f embed go`：Go TUI 编译产物嵌入 TS 包发布。**Go 没有被删，被降级为一个包**——真正的删除要到 2025-11-02 `f68374ad22 DELETE GO BUBBLETEA CRAP HOORAY`（commit 标题本身就是最好的史料）。

### 2.5 架构对比：从单体到 core/client（考古实证的目录结构）

```
Go 原型（05-16，d0ad09d798 时点）            TS 重写后（06-03，caf9fdc893 时点）

┌───────────────────────────┐                ┌───────────────────────────────┐
│        termai/opencode    │                │   packages/opencode  (core)  │
│  ┌─────────┐ ┌─────────┐  │                │  app/ bus/ config/ global/   │
│  │   TUI   │ │  LLM    │  │                │  id/ lsp/ permission/        │
│  │ (bubble │ │ agent + │  │   ──重做──▶     │  provider/ server/ session/  │
│  │  tea)   │ │ tools + │  │                │  share/ storage/ tool/       │
│  │         │ │LSP(6 个)│  │                └──────────┬────────────────────┘
│  └─────────┘ └─────────┘  │                           │ OpenAPI 契约 (gen/)
│   pubsub 事件总线贯穿其中   │                           ▼
└───────────────────────────┘                ┌───────────────────────────────┐
  进程内函数调用，无 server 层                │ packages/tui (Go bubbletea)   │
                                             │ pkg/client/generated-client.go │
                                             └───────────────────────────────┘
                                              降级为 HTTP 客户端 + 渲染端
```

模块级映射（两侧目录均为 `git ls-tree` 实证）：

| Go 原型（internal/） | TS core（packages/opencode/src/） | 演化说明 |
|---|---|---|
| `pubsub/broker.go` | `bus/` | 事件总线原样继承，胎记保留 |
| `llm/tools/*.go`（17 个） | `tool/*.ts`（11 个） | 语义级移植，LSP 工具收敛 |
| `llm/models/` + `llm.go` | `provider/` | provider 层抽象化、可配置化 |
| `permission/` | `permission/` | 逐字对应，产品核不动 |
| `lsp/`（含 watcher 633 行） | `lsp/` | 移植 |
| —（无） | `server/` | **全新**：HTTP + OpenAPI，架构反转的支点 |
| —（无） | `share/` | **全新**：share page，与 SST app 同期 |
| —（无） | `storage/` | **全新**：会话持久化独立成层 |

这张表就是"重做而非翻译"的第三条证据：三个全新目录（server/share/storage）恰好都是"只有 core/client 架构才需要"的东西。

## 3. 为什么弃船（三个假说与证据强度）

> 本节为**分析**级：弃船动机无作者公开陈述，以下为证据强度评估。

**假说一：终端 UI 组件化前景 —— 证据强度：中强。**

考古证据：首 commit 40% 的代码是 TUI 布局/样式；4 月的提交几乎全是 UI 打磨（diff 着色、动态对话框、状态栏）；`wip: refactoring tui` 在重写期反复出现——TUI 是 Go 版最大的维护负担。反向证据：最终方案里 Go bubbletea 又活了 5 个月（05-30~11-02），说明"终端渲染"本身不是死因，**死因是"用 Go 写业务组件"**——一旦 TUI 降级为纯渲染客户端，Go 就只剩渲染价值，而渲染层恰好是可以最后替换的。四端同构需求（Solid 渲染终端）在 Go 版里无解。

**假说二：插件/动态语言需求 —— 证据强度：中。**

直接证据不足：plugin 系统和 Codemode JS 解释器都是弃船之后数月才出现的产物。但间接证据存在：配置文件在 04-16 rename commit 已是 `.opencode.json`（JSON schema），TS 化后配置可直接演进为可编程的 `opencode.jsonc` + 后来带函数的 TS 配置；`example.ts` 工具在 TS core 首月就出现（"工具即代码"的雏形）。定性：**更像是 TS 化的收益被后见之明放大，而非当初的主因**。

**假说三：SST 生态协同 —— 证据强度：强。**

最硬的证据链：Dax Raad（SST 作者）05-17 亲自下场写第一个 TS commit；`app/`（SST infra）05-23 进仓库，**与 share page 同期出现**——share URL 需要 serverless 后端，SST 是 Dax 的自家武器，Go 版做 share page 要另起一套部署体系。05-30 大重组把 `sst.config.ts` 提到仓库根，monorepo 完全 SST 化。时间因果清晰：share 功能（05-23）→ 重组（05-30）。这个假说唯一弱点是"share 也可用 Go 部署"，但生态成本对比悬殊。

**综合（分析）**：假说三是最可能的直接导火索（人 + 功能双双到位），假说一是结构性主因（TUI 组件化天花板），假说二是事后兑现的红利。八周弃船在个人项目里不算仓促——恰恰是原型期跑通了"agent + 工具 + 权限"的产品核（这部分全部语义级移植到了 TS），才敢弃船；而 4 月已成形的社区（18 个贡献者）则是敢于把 Go 版降级但保留为客户端的约束条件。

## 4. 决策的长期回响

只有 TS 能做、或 TS 后才做出来的东西（按出现顺序）：

```
2025-05-30  monorepo 化 ── core(TS) + tui(Go 客户端) + web/share(SST) 三包同仓
2025-06     OpenAPI 契约 ── 任意语言客户端可生成（Go TUI 即首个消费者）
2025 下半年  plugin 生态 ── 用户以 JS/TS 写插件，宿主同语言，零 FFI
             Codemode ──── JS 解释器沙箱（模型生成的代码直接跑，Go 需嵌 wasm/quickjs）
2025-11-02  f68374ad22 "DELETE GO BUBBLETEA CRAP HOORAY" ── Solid 渲染终端，四端同构
             （desktop/web/tui/... 共享 Solid 组件树——Go 版架构上不可能）
```

关键回响是**架构反转被证明可复用**：05-30 确立的 "TS core + 契约 + 多前端" 让 11 月删 Go TUI 时 core 零改动——bubbletea 时代抱怨的 charm 性能问题（`4699739814 shitty hack for terrible charm bubbletea performance`，2025-07-19）才有了出路。若当年留在 Go，四端同构与插件生态两个叙事都不成立。

另一个回响是组织性的：重写确立了 "Dax 主导架构、Kujtim 主导产品、社区跟进" 的分工模式，5 月的 commit 作者分布（adamdottv 120 / Dax 102 / 其他）就是这个模式的第一张快照。

## 时期小结

一句话定性：**termai 是一次用 Go 快速验证"agent + 工具 + 权限"产品核的原型实验，八周跑通后被整体降级为 TS monorepo 里的一个渲染客户端——弃的是 Go 写业务的天花板，留下的是已验证的产品骨架与契约优先的架构。**

---

### 复现命令清单

```bash
git show 4b0ea68d7a --stat                        # 原型骨架
git show 4b0ea68d7a:go.mod                        # charm 全家桶选型
git show afd9ad0560:go.mod                        # 双 LLM SDK
git show 904061c243 --stat                        # 工具一夜成型
git show cfdd687216 --stat                        # LSP 大 commit（+13991）
git log --reverse --oneline | head -30            # 早期轨迹
git log -1 --format='%h %ad %s' --date=iso cc07f7a186   # rename to opencode
git log --format='%an' --since=2025-04-01 --until=2025-04-30 | sort -u  # 4 月贡献者
git log --reverse --oneline -- js/                # TS 首现 05-17
git log --oneline --diff-filter=D -- go.mod       # 05-30 根 go.mod 删除
git show f3da73553c --stat                        # 大重组
git ls-tree --name-only caf9fdc893:packages/opencode/src/tool/   # TS 工具面
git ls-tree --name-only d0ad09d798:internal/llm/tools/           # Go 工具面
git log -1 --format='%ad %s' --date=iso f68374ad22               # DELETE GO BUBBLETEA CRAP HOORAY
```
