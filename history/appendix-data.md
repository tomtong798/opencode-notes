# 数据附录（Data Appendix）

> 本卷全部原始证据，2026-10-22 采集。全部可用文中命令复现。

## 1. 基本盘

| 指标 | 值 | 采集方式 |
|---|---|---|
| 仓库 | anomalyco/opencode（本地 remote 实证） | `git remote -v` |
| Stars | 219,830 | GitHub API `/repos` |
| 建 GitHub 仓库 | 2025-04-30 | 同上 `created_at` |
| 首 commit | 2025-03-21（`4b0ea68d7a initial`，Go：`cmd/termai/main.go` + `go.mod`） | `git log --reverse` |
| 总 commits | 20,483 | `git log --oneline \| wc -l` |
| 总 issues（含 PR） | 31,134 | search API `total_count` |
| issue/PR 比 | 建库初期 7 周：723 issues / 277 PRs | REST API 全量前 10 页 |

## 2. 月提交量

```
2025-03        8    2025-11     1049    2026-06     1127
2025-04      152    2025-12     1877    2026-07     1601
2025-05      301    2026-01     2083    2026-08     1984
2025-06      654    2026-02     1165    2026-09     1486
2025-07      650    2026-03      963    2026-10      663
2025-08      614    2026-04     1328    （月中）
2025-09      518    2026-05     1551
2025-10      709
```
命令：`git log --format="%ad" --date=format:"%Y-%m" | sort | uniq -c`

## 3. 关键时间点编年（git 考证）

| 日期 | 事件 | 证据 |
|---|---|---|
| 2025-03-21 | Go 原型 termai 首提交 | `git log --reverse` 首 8 commits |
| 2025-04-16 | 改名 opencode | commit `rename to opencode` |
| 2025-05-30 | **go.mod 删除，TS 重写完成** | `git log --diff-filter=D -- go.mod` |
| 2025-07-07 | 第一代 "v2 message format"（AI SDK v5 升级） | PR #743 |
| 2025-07-22 / 08-02 | sdk / plugin 包诞生 | `--diff-filter=A -- packages/*/package.json` |
| 2025-09-15 / 10-03 | app / desktop 包诞生 | 同上 |
| 2025-12-07 | "sdk v2"（第二代 v2 概念试验） | PR #5216 |
| 2025-12-10 | global bus（V1 内局部引入总线） | commit `global bus` |
| 2026-02-02 | 社区提出 Ink-based TUI Rewrite | issue #11762 |
| 2026-03-12 | **effect migration plan 出现** | commit `keep effect migration plan local` |
| 2026-03-18 | Bus 迁移 Effect | commit `migrate Bus to Effect service` |
| 2026-04 下旬 | httpapi parity 契约测试网铺开 | `cover full OpenAPI route inventory` 等 |
| 2026-04-15 / 04-25 | **server / core 包诞生（V2 动工）** | package.json diff-filter=A |
| 2026-05-30 | core/src/event 诞生（事件溯源起点） | 同上 |
| 2026-06-03 | runner + run-coordinator + skill 落地；同日 tui 包 + **OpenTUI** 诞生 | 同上 |
| 2026-06-24/25 | **schema / protocol / client 契约层诞生** | 同上 |
| 2026-07-06 | durable compaction barrier / durable instruction | commit log |
| 2026-07-09 | execution/restart 恢复子系统 | diff-filter=A |
| 2026-07-19 | tui/mini 第二界面 | 同上 |
| 2026-07-20 | **V1 主入口删除（V1 退役）** | `--diff-filter=D packages/opencode/src/index.ts` |
| 2026-07-26 | bus.ts 定型 / 工具架构合并 | commit log |
| 2026-08-11 | write-ahead execution claims | PR #41800 |
| 2026-08-12 | generic session inbox（准入/执行分离） | PR #42109 |
| 2026-09-11 | Windows CLI 签名（#631 终章） | release commits |
| 2026-10-07 | plugin/runtime 插件宿主定型 | diff-filter=A |

## 4. V1 膨胀证据

`packages/opencode/src` 文件数走势（`git ls-tree -r` 于各时点）：

```
2025-08-01    117 文件
2025-11-01    195 文件
2026-02-01    332 文件
2026-04-01    437 文件（峰值；2026-07-20 退役）
```

V1 时期（2025-07~2026-01）fix 提交主题词频：

```
tui 110（含 TUI 13）│ session 63 │ provider 27 │ scroll 26 │ render 20
plugin 16 │ windows 25（含 Windows 11）│ lsp 28（含 LSP 14）│ mcp 9
```
命令：`git log --format="%s" --since=... --until=... | grep -iE "^fix" | grep -oiE "..." | sort | uniq -c`

## 5. 建库初期（2025-04-30~06-19）issue 全量画像

早期标签分布：`growth 6 / help-wanted 5 / bad 4 / model-problem 4 / windows 2 / bug 1 / perf 1`（标签体系当时尚不成熟）。

最早 20 个 issue 节选（雏形问题面）：

```
#2  Roadmap & Existing Issues          #17 Override Default Keymaps (💬9)
#3  model config persistence           #19 custom themes 0-255 (💬4)
#7  Interactive Configuration (💬5)     #20 shift+enter newline (💬4)
#9  full-auto / permissions (💬6)       #27 codex-mini, responses api (💬9)
#11 ollama support (💬5)                #31 Denying tool continues generation
#6  cant install (💬3)                  #33 openrouter multi-model broken
#10 diff rendering issues              #41 Interrupt breaks new messages
```

初期讨论量 TOP：

```
#631  Windows Support                          💬207
#811  Text rendering VERY slow + CPU high      💬84
#239  Implement permissions in TUI             💬67
#988  MCP remote oauth                        💬42
#531  HTTP_PROXY support                      💬40
#266  gemini doesn't handle edit tool well    💬39
```

## 6. 全期反应数（👍）TOP 榜（31,134 issues 抽样）

```
#7410  Broken Claude Max                     👍420  closed
#16992 /btw command                          👍410  closed（已实现——composer 走读里的 /btw 分流）
#8501  展开粘贴文本折叠                        👍328  closed
#1686  OpenAI auth login like claude          👍271  closed
#2072  Support for Cursor?                   👍264  closed
#6231  Auto-discover OpenAI-compat models    👍261  closed
#27167 /goal native session goals            👍216  open（fork 的 06-goal 对应此需求）
#4695  Speech-to-Text voice input            👍208  closed
#1764  vim motions in input box              👍198  closed
#12661 Agent Teams equivalent                👍190  closed（subagent 机制回应）
#20695 Memory Megathread                     👍173  closed（fork 的 09-memory 对应）
#11176 Official VS Code extension            👍175  open
#631   Windows Support                       👍137  closed（16 个月长跑）
#3235  Support for "Skills"                  👍132  closed（skill 子系统回应）
#4821  unqueue messages                      👍126  closed（inbox cancel 回应）
```

## 7. 痛点 → 机制对照（社区困难清单与代码机制清单的镜像）

| 痛点 issue | 沉淀为的机制 | 笔记落点 |
|---|---|---|
| #9/#239 权限争议 | permission 系统（默认 ask 白名单 + deny 前置） | 6_modules/11_permission |
| #811/#265 渲染慢 | OpenTUI 自研渲染器（Solid→终端） | 2_tech-stack/3_opentui |
| #11/#33/#469 模型碎片 | models.dev 目录 + @opencode/ai 统一抽象 | 6_modules/3_model-protocol |
| #631 Windows | 三平台 #sqlite / wasm 解析器分平台 / 签名流水线 | 7_nonfunctional 各篇 |
| #41/#31 中断语义 | Effect interruption 贯穿 + resume:true | 6_modules/4_session |
| 崩溃丢状态（反复出现的重启类 bug） | durable 事件 + inbox 准入 + claim 恢复 | 7_nonfunctional/1_durability |
| #988 MCP oauth | mcp/oauth.ts 轮换去重 + loopback | 6_modules/14_mcp |
| #4821 unqueue | inbox cancel/delivery.changed | 6_modules/5_inbox-coordination |
| #20695 memory 长跑 | 上游未定论；fork 自研 09-memory | features/ |

## 8. Agent 技术演进数据（第 5 篇专用）

**工具首现时间线**（`git log -i --grep` 首次命中）：

```
2025-04-04 bash 工具（同期还有权限对话框 UI 增强）
2025-04-14 edit 工具
2025-05-01 task 工具（subagent 雏形）
2025-05-02 compact 命令 + auto-compact
2025-05-27 grep 工具
2025-06-04 todo 工具
2025-06-05 webfetch 工具
2025-06-12 subagent 首批提交
2025-06-23 glob 工具
```

**agent 关键 commit 编年**：

```
2025-03-23 initial agent setup（Go 期）
2025-06-15/16 system prompt 调优开始（"less fast" / 缓存限 4 份）
2025-07-15 plan mode enable bash；08-08 移除 plan mode synthetic message
2025-07-19 disable todo tools when running as task（子代理工具面裁剪的最早发现）
2025-07-31 wildcard matching improve（权限第一轮）
2025-12-31 + 2026-01-01 Permission rework ×2（#6319 大改）
2026-01-01 check context overflow mid-turn（#6480）
2026-04-08 task tool from agent services（registry 化起点）
2026-04-29 small model for subagents / limit small models to explore agents
2026-05-09 ★subagent inherits parent deny rules（Plan Mode security bypass #26597）
2026-05-12 preserve subagent self permissions（#27201）
2026-06-03 skill/discovery 落地；06-20 honor agent step limits（#33142）
2026-06-30 yolo permission mode（#33279）
2026-07-12 remove tool concurrency cap（codemode #36545）
2026-08-14 minimize system prompt（#42638）
2026-09-03 command subagents in background（#47081）
2026-09-15 model identity appended to system prompts（#49129）
```

**edit 工具可靠性**：`--grep="edit tool"` 命中 27 个提交，2025-06-25 同日两连 "make edit tool more robust"——最难做对的工具。

**系统提示词反复实验**：2026-07-27 Meta prompt 三连 fix、08-10 Muse 路由、09-08 Astra port、09-11 Anthropic comment guidance。

**agent 技术侧高反应 issue**：

```
形态：#12661 Agent Teams 👍190(open) │ #12711 flat teams 设计 👍74(open)
      #8751 热重载 agents/skills 👍114 │ #5887 异步后台子代理 👍99
      #2242 sandbox the agent 👍87(open)
上下文：#6152 context usage 显示 👍154 │ #8625 MCP 占上下文 👍85
        #8140 可配置压缩阈值 👍82 │ #4659 滑动窗口 👍46 │ #11314 压缩阈值 👍36
        #6316 AGENTS.md 按需加载 👍29 │ #11995 工具描述吃 token 👍17
        #29672 提示词提缓存命中 👍15 │ #23595 system-reminder 毁缓存 👍15
上下文窗口事故：#152 窗口警告(2025-05-09) │ #5480 LSP 诊断限流(2025-12-14)
              #6480 mid-turn 溢出检查(2026-01-01)
```

## 9. 采集方法与限制（诚实声明）

- git 数据：本地仓库全史，命令可复现，**实证**级；
- 建库初期 issue/PR：REST API 前 1,000 条全量（page>10 需 cursor 分页，已处理），**实证**级；
- 全期榜单：search API 按 reactions 排序取 TOP100，非全量遍历（GitHub 限流 60 req/h 未认证），榜单名次**实证**、尾部可能遗漏；
- "为什么放弃 Go"等动机推断：无官方文档佐证，基于后续技术选择的**分析**级判断，已在正文标注。
