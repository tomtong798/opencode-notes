# Agent 技术演进：从硬编码到机制化（贯穿四时期的第五线）

> 基线：git + issue 数据（appendix-data.md §8/§9 同步扩充）| 实证 + 动机分析标注
> 复现前提：`cd /home/donghe/github/opencode` 后逐条执行文中命令。本篇是历史卷第 5 篇——
> 不讲架构骨架（那是 1-4 篇的事），讲"智能体能力本体"的工程化痛史：agent 定义、工具、
> 上下文、权限、多智能体。回答一个质疑：**历史卷只写了架构调整，agent 技术本身的痛点呢？**

---

## 0. 时期速览（四阶段图）

```
agent 能力四阶段（横向 = 时间，与历史卷四时期对齐）

【1】Go 原型        【2】TS 重写/雏形     【3】V1 膨胀期          【4】V2 重建
2025-03~05          2025-05~07           2025-07~2026-03        2026-03~10
   │                   │                    │                      │
 单 agent 硬编码     工具丰富化           plan/subagent 分工      机制化
 internal/llm/agent  bash→edit→task→      7 agent 硬编码清单      registry/plugin/
 一个 loop 跑一切    grep→todo→webfetch   + subagent 生态补丁     background/inheritance
   │                   │                    │                      │
 e7258e38            49423da0 compact      #1689 agent∩mode       #21017 task←services
 "initial agent      2025-05-02 auto      2025-08-07 合并         2026-04-08 registry 化
  setup"             a454ba88 subagent     8f64c4b3 子代理裁工具   #26597 deny 继承
 2025-03-23          2025-06-12 首提交     2025-07-19             2026-05-09
```

一句话总纲：**agent 能力的每一步都不是设计出来的，是被工具失败、上下文爆炸、权限绕过、
子代理失控一次次逼出来的。** 架构线（1-4 篇）回答"房子怎么改"，本篇回答"住户怎么学会生活"。

---

## 1. Agent 形态：从 7 个硬编码到编排原语

### 1.1 起点：一个 loop 的时代（Go 期）

```
2025-03-23 e7258e38ae "initial agent setup"   ← internal/llm/agent/，Go
2025-03-24 005b8ac167 "initial working agent" ← 加 title 生成，还是单 loop
2025-04-16 bbfa60c787 "reimplement agent,provider..."
```

Go 期的"agent"就是一个 while loop：发消息→执行工具→回填→再发。没有 plan/build 之分，
没有子代理，工具只有三四个（bash 04-04、edit 04-14）。**形态即架构**——单文件单 loop，
agent 一词尚未成为独立概念。（实证：`git log --reverse -i --grep=agent | head`）

### 1.2 七agent 硬编码清单：V1 的"形态定型"

TS 重写后 agent 概念落地为一份硬编码清单。在 V1 峰值（2026-03-31，`c8ecd640`）实测
`packages/opencode/src/agent/agent.ts`：

```
build      mode: primary    ← 干活的
plan       mode: primary    ← "Plan mode. Disallows all edit tools."
general    mode: subagent   ← 通用子代理
explore    mode: subagent   ← 只读探索（2025-11-28 才加入，#4893 时代）
compaction mode: primary    ← 压缩专用
title      mode: primary    ← 标题生成（small model 的前身概念）
summary    mode: primary    ← 摘要
```

7 个 agent，全写死在代码里，用户只能覆盖不能增删结构。两个观察：
- **内部机制 Agent（compaction/title/summary）和面向用户 Agent（build/plan）混在同一清单**——
  "agent 即配置单元"的观念尚未诞生，什么都是 agent，等于什么都不是；
- 期间还发生过一次概念合并：2025-08-07 `c34aec060f` "Merge agent and mode into one (#1689)"——
  早期 mode（plan/build）与 agent 是两个概念，合并后才统一为 agent 定义。

### 1.3 plan mode 的 enable/disable 反复：形态试错的标本

plan（只读规划）与 build（可写执行）的边界，是 V1 期最反复的一条线（全部实证）：

```
2025-07-15 05bb127a8e "enable bash tool in plan mode"        ← diff 仅 -1 行（mode.ts）
2025-08-08 7bbc643600 "remove synthetic message in plan mode,
                       fixes being confused in build mode"    ← 合成提醒消息把模型搞糊涂了
2025-08-11 457386ad08 "fix plan mode bash tool making changes" ← 给了 bash 又管不住
2025-08-19 25f43adaa0 "notify agent it is in build mode when
                       switching from plan mode (#2065)"
2025-08-21 ba2e86c7ef "adjust plan agent to ask when running bash,
                       give it edit tool… (#2150)"            ← 又把 edit 给回去了
2025-09-08 162a789fa2 "remove edit tool from plan agent (#2505)" ← 又拿走了
2025-10-24 4c90bf3e07 "whitelist some safe bash tools in Plan agent (#3288)"
            e88b659545 "make plan agent whitelist more conservative (#3424)" ← 同周收紧
2026-01-13 3997d3f2d7 "feat: add plan mode with enter/exit tools (#8281)"  ← 终局：
                                                                        plan 变成模型可调用的工具
2026-05-13 098bdd8ae2 "route plan mode through runtime flags (#27320)"
2026-08-21 8b93bc395d "allow plan mode to write/edit PLAN files
                       exclusively (#43710)"                  ← 写权限按文件白名单回归
2026-08-11 a5d6969f6f "extract plan agent into a plugin (#41665)" ← V2:插件化
```

这条线的读法：**"只读 agent"不是一个声明就能成立的属性，而是工具面、提示词、运行时拦截
三层反复对齐才勉强稳定的状态。** 给 bash→越权→收白名单→太紧→再松，八个月里钟摆了六个来回，
最终解法不是"调对了参数"，而是把 plan/build 切换本身降格为模型可调用的工具（enter/exit），
让形态问题变成机制问题。（"钟摆源于规划型任务对环境信息的真实需求"——分析级推断）

### 1.4 explore 与 small model：分工的经济学

```
2025-11-28 813d287a09 "core: add explore agent for fast codebase
                       navigation and improve task UI display"
2025-11-30 ff609a52c1 "docs: explore subagent (#4893)"
2026-01-13 bcdaf7e779 / 0eb898abcf "tweak: prompt for explore agent better"（同日两次）
2026-02-17 25f3eef957 "ensure explore subagent has external_directory perm
                       set to ask instead of auto denying (#14060)"
2026-04-29 6ba2bbc91c "feat(task): use small model for subagents"
2026-04-29 ed96212712 "fix(task): limit small models to explore agents"
2026-10-05 7725ac81eb "feat(core): allow shell for explore (#53284)" ← 一年后又给 shell
```

2026-04-29 同日一放一收（explore 用小模型、但小模型只准用于 explore），是"成本分层"思路
成型的标志：探索是高频低难任务，配便宜的脑子；但小模型干不了 build，必须圈住。
（动机：token 成本与延迟——分析级；与 6_modules/8_agent 的 Step Allowance 机制衔接。）

### 1.5 未解决的渴望：Agent Teams

社区对"更高阶编队"的呼声至今是 agent 形态线的悬案：

```
#12661 Agent Teams equivalent            👍190（官方后续以 subagent 机制回应）
#12711 flat teams 设计讨论               👍74
#8751 热重载 agents/skills/commands      👍114
#15475 plan mode 换 build agent 执行     👍21
```

opencode 的答案长期是"**不引入 Team 原语，只给 task 工具 + 后台作业**"（见 §5），
Teams 派用户始终认为形似神不似。**未解决的渴望也是历史**——这条线的存在本身说明：
单父多子的树形编排在真实工作流里不够用，而 opencode 宁可把复杂性留给机制演化（V2 的
subagent-job/completion）也不肯过早固化 Team 语法。（取舍判断为分析级）

---

## 2. 工具工程化：补全史与 edit 的 27 连修

### 2.1 工具补全时间线（实证，`git log -i --grep` 首次命中）

```
2025-04-04 bash   （Go 期；同日 4 个权限对话框 UI commit——工具与权限同时出生）
2025-04-14 edit
2025-04-27 glob 前身（ripgrep-glob 分支，性能动机）
2025-05-01 task   ← subagent 雏形工具先于 subagent 概念两周
2025-05-02 compact 命令 + auto-compact（上下文自救第一天，见 §3）
2025-05-14 docSymbols/workspaceSymbols（LSP 工具，后被裁撤）
2025-05-26 ls tool
2025-05-27 grep
2025-06-04 todo
2025-06-05 webfetch（06-10 "spoof browser"——反爬逼出来的 UA 伪装）
2025-06-23 glob（respect .gitignore）
```

两个月内七个核心工具全部到位，之后两年只是打磨。注意顺序的合理性（分析）：执行（bash）
→ 修改（edit）→ 委派（task）→ 自救（compact）→ 检索（ls/grep/glob）→ 记忆（todo）→
外联（webfetch）——**按"单 agent 的生存刚需"排序补全，多智能体排最后**。

### 2.2 edit 的 27 连修：最难做对的工具

`git log --oneline --all -i --grep="edit tool" | wc -l` = **27**。密度最高的一天：

```
2025-06-25 fc7af31fe5 make edit tool more robust
2025-06-25 66830ced4e make edit tool more robust        ← 同日两连
2025-06-25 d240f4c676 more edit tool fixes
2025-06-25 fab8dc9e6f more edit tool fixes              ← 同日又是两连
2025-06-25 9c90cdbe08 integrate gemini-cli strategies for edit tool
2025-06-25 f39a2b1f16 integrate gemini-cli strategies for edit tool（重复落地）
```

单日 6 个 commit，包括把 gemini-cli 的容错策略整个搬来。为什么 edit 这么难？
长尾证据给出三个答案：

```
#266  gemini doesn't handle edit tool well（2025-06，💬39）  ← 模型侧适配
2025-09-05 #2051 "separate edit tool error message with clearer
            guidance to avoid llm doom editing loop"        ← 失败循环：改不对→再改→更错
2026-10-12 #53166 "preserve mixed line endings in edit tool" ← 18 个月后还在修换行符
```

（判断，分析级：edit 的难度是三体问题——字符串匹配的脆弱性 × 模型指令遵循的方差 ×
真实文件的脏数据（CRLF/空白/编码）。bash 失败会报错给用户，edit 失败会被模型"自愈"成
更糟的补丁，所以必须连"报错措辞"都要设计——#2051 修的是错误文案。）

### 2.3 "disable todo when as task"：子代理工具面裁剪的最早发现

```
2025-07-19 8f64c4b312 "disable todo tools when running as task"   ← 子代理不该有 todo
2025-07-25 d6b3bb0807 "disable todo tools by default in agent"    ← 默认 agent 也关掉
2025-08-12 5cc44c872e "disable todo tools for qwen models"        ← 模型能力适配
2026-01-19 3515b4ff7d "omit todo tools for openai models"
2026-07-09 7feefb697f "refactor: remove todo tool (#35989)"       ← todo 工具本体退役
```

这条线的演化方向很有意思：从"按身份裁剪"（task 中禁用）→"按默认裁剪"（默认关闭）→
"按模型裁剪"（qwen/openai 不配）→ 最终 todo 走向 todo dock UI + todowrite 服务
（2026-04-03 `f2d4ced8ea` 从 Todo service 构建 todowrite 工具），裸 todo 工具删除。
**工具不是加法史，是裁剪史**：每加一个工具都要回答"谁在什么模型上配不配用它"——
这是工具工程化成熟的标志，也是 §5 权限继承问题的前奏。

### 2.4 工具描述与系统提示词：token 经济学

```
#11995 工具描述吃 token（👍17）
2026-05-16 #26821 "core: reduce prompts for shell, todowrite, and task tools"
2026-07-25 #38899 tweak glob tool description/parameters
2026-08-14 #42638 "minimize system prompt"
```

系统提示词的"反复实验"更长（实证节选，完整清单见 appendix §8）：

```
2025-06-15 7d174767b0 "first pass making system prompt less fast" ← 标题即吐槽
2025-06-16 63996c4189 "limit to 4 system prompts cached"
2026-02-03 #12025 Trinity prompt 加入 → #12029 同日 revert       ← 加了又删
2026-03-26 #19220 gpt 专用 prompt / 03-31 #20259 Kimi 专用
2026-07-09 #36122 Meta system prompt 加入
2026-07-27 #39237/#39240/#39245 Meta prompt 三连 fix（同日）
2026-08-10 #41581 Muse 路由到 Meta prompt
2026-09-08 #48057 Astra port / 09-11 #48444 Anthropic comment guidance
2026-09-15 #49129 model identity appended to system prompts
```

主线（分析级）：系统提示词从"一份通用文档"演化为"**按模型路由的插件化提示词族**"
（2026-07-15 `720f062cd0` "select system prompts through plugins"），动力有二：
不同模型吃不同措辞（#266 教训），以及提示词体积直接吃上下文/缓存（#11995/#29672）。
机制细节不再展开，见 6_modules/6_context §2。

---

## 3. 上下文管理：从报警到精细治理

### 3.1 事故编年（全部实证）

```
2025-05-02 49423da081 "feat: compact command with auto-compact"  ← 与 webfetch 同期的自救
2025-05-09 #152 Context Window Warning                            ← 第一声报警（雏形期）
2025-06-20 194415e785 footer 显示 context usage                   ← 可见性第一步
2025-09-13 a00b49d65b disable autocompact if context is 0
2025-09-16 3aeac02bf1 enable session pruning（滑动窗口雏形，#4659 的回应）
2025-09-17 259c722208 only prune messages from more than 2 turns ago
2025-12-14 #5480 "limit LSP diagnostics to prevent context window waste"
2025-12-26 ed06de5e30 可配置 compaction settings（#8140/#11314 的回应）
2026-01-01 #6480 "check for context overflow mid-turn in finish-step"
2026-01-26 #10678 AGENTS.md 子目录按需加载（#6316 的回应）
2026-02-02 #11581 防止 AGENTS.md 重复注入
2026-03-02 #14707 413 错误经 auto-compaction 自恢复
2026-08-11 #41652 AGENTS.md 向上发现到 home 目录
```

### 3.2 弧线：报警 → 可见 → 限流 → 恢复 → 治理

把高票 issue 排在事故线上，能看到一条完整的治理弧线：

```
阶段              标志事件                        对应社区痛点
─────────────────────────────────────────────────────────────
Ⅰ 报警     #152 窗口警告(2025-05)           "炸了才知道"
Ⅱ 可见     footer usage(06-20) / #6152 👍154 "看不到还剩多少"
Ⅲ 限流     #5480 LSP 诊断限流(2025-12)       "工具输出偷吃窗口"
           #11995 工具描述吃 token 👍17
           #8625 MCP 工具占上下文 👍85
Ⅳ 恢复     #6480 mid-turn 溢出检查(2026-01)  "半路炸了怎么办"
           #14707 413 → auto-compaction
Ⅴ 治理     压缩阈值可配置(#8140 👍82/        "让我自己调"
           #11314 👍36)、AGENTS.md 按需加载
           (#6316 👍29)、缓存命中优化
           (#29672/#23595 各 👍15)
```

两个值得驻足的细节：
- **#5480 的教训**：LSP 诊断曾是 edit 工具的"贴心"回显，直到有人发现诊断文本本身
  在偷吃窗口——**帮 agent 的信息也可以害 agent**。同一条思路后来用于工具描述瘦身
  （#26821）和 minimize system prompt（#42638）。
- **mid-turn 检查（#6480）**：溢出不发生在"你发消息时"而发生在"模型第 40 步工具调用后"，
  所以检查点必须挂在 finish-step 里——上下文治理从请求边界下沉到了 step 边界，
  这正是 V2 compaction 三路触发（见 6_modules/7_compaction §2）的前身。

### 3.3 缓存命中：上下文治理的隐藏战线

```
2025-06-16 0e3458b112 fix cache-control / 06af406146 track cache token counts
2025-12-15 #5550 rejoin system prompt … to preserve caching   ← 提示词拼接顺序毁缓存
#23595 system-reminder 移动毁缓存（👍15）
#29672 改提示词提缓存命中（👍15）
2026-09-28 #51960 drop session ID / order instructions for prompt-cache reuse
2026-09-21 #50495 share child prompt cache affinity            ← 子代理共享父缓存亲和
2026-10-12 #54945 Anthropic cache breakpoints 可配置
```

缓存这条线说明：上下文管理不仅是"别超"，还有"别变"——**任何动提示词顺序的优化
（哪怕是修 bug）都可能让整段缓存失效**。#5550 和 #23595 是同一教训的两次学费。

V2 的终点：compaction barrier / durable instruction 三表（2026-07-06 落地）把"哪些
指令必须幸存于压缩"变成数据结构——这条从 #152 报警出发的线，走了 14 个月到达机制化。
不再重复 6_modules/6_context（指令状态三表）与 7_compaction（压缩执行流）已覆盖的细节。

---

## 4. 权限与安全：两代 rework 与一次绕过事故

### 4.1 编年

```
2025-04-04 Go 期：bash tool security + 权限对话框 UI ×4     ← 工具出生当天就有权限
2025-05-16 #25 权限窗口显示错误路径
            #31 Denying tool continues generation            ← 拒绝后模型继续跑，早期语义混乱
2025-06-02 786db364d2 "add permission system"               ← TS 期权限系统正式落地
            #9 full-auto / permissions（💬6）/ #239 TUI 权限（💬67）← 早期最大争论
2025-07-31 wildcard matching improve（+ wip:permissions ×4） ← 第一轮打磨
2025-08-05 #1627 disallow / 08-10 #1772 webfetch permission
2025-08-12 #1862 agent 级权限白/黑名单
2025-08-13 #1879 默认 agent 权限与全局 config 合并           ← 两层权限怎么叠加，第一天的问题
2025-09-14 #2592 bash 权限按命令模式匹配
2025-12-31 → 2026-01-01 #6319 "Permission rework"           ← 第一代大改（跨年提交）
2026-06-30 #33279 yolo permission mode                       ← 全放行的官方出口
2026-07-12 #36545 remove tool concurrency cap（codemode）
2026-09-17 fa126d68e8 "enforce permission policies"          ← V2 策略执行
2026-10-13 #54930 plugins assert permissions                 ← 插件介入权限
```

### 4.2 #6319 rework 的实体

第一代 Permission rework（#6319，2026-01-01）diff 实测：新增 `permission/arity.ts`（163 行）、
`permission/next.ts`（253 行）、**652 行权限测试**——重写的不是 UI 而是求值语义
（ask/allow/deny 的优先级与 arity 合并）。#9/#239 争论的"full-auto 到底多 full"，
最终答案分两路：语义上靠 rework 的确定性求值，产品上靠 2026-06 的 yolo 模式
（#33279）——**与其让用户在 issue 里吵"能不能全放行"，不如给全放行一个明确的名字**。
（yolo 的产品化判断为分析级；社区呼声为实证。）

### 4.3 ★Plan Mode 安全绕过事故（#26597）：权限继承的里程碑

```
2026-04-30 #23290 preserve external_dir & deny parent permissions in task child
2026-05-09 #26597 ★subagent inherits parent agent's deny rules
                (Plan Mode security bypass)
2026-06-10 #31696 let subagents use their own permissions      ← 继承与自治的再平衡
2026-06-30 c26f6f95f7 deny subagent tool for default subagents ← 防递归委派越权
2026-08-20 #43675 answer subagent permissions in run
2026-08-31 #44976 surface subagent permissions → #46376 同月 revert
```

事故机制（实证 + 分析）：plan 模式靠 deny 规则禁写，但 task 工具派生的子代理**默认不继承
父 agent 的 deny**——模型在 plan 模式下可以"委派一个 build 子代理"绕过只读约束。
修复 `b8ca71d309`（2026-05-09）新增 `subagent-permissions.ts`（33 行）并附带
**141 行回归测试** `plan-mode-subagent-bypass.test.ts`——这是本卷观察到的第一次
"安全修复自带场景级回归测试"。

里程碑意义（分析级）：它揭示多智能体系统的权限不是"每个 agent 一份配置"，而是
**继承语义**——父的 deny 必须不可逃逸，子的 allow 可以自主。此后的 #31696（子的自身
权限不被父覆盖吞掉）、c26f6f95f7（默认子代理禁用 task 工具防递归放大）都是同一语义
的补全。V2 的权限内嵌模型见 6_modules/11_permission 与 8_agent §7。

### 4.4 未了：sandbox

```
#2242 sandbox the agent（👍87，长期 open）
2026-07-13 #36768 Sandbox 术语更名为 CodeMode
2026-07-12 #36545 codemode 移除工具并发上限
2026-08-07 #41177 modal sandboxes on vm runtime
```

权限是"问过了才许做"，sandbox 是"物理上做不了"——后者至今仍是 open 的渴望，
V2 期只走到 CodeMode（宽松并发 + vm runtime 隔离执行）这一步。（进展归因：分析级。）

---

## 5. 多智能体：task 工具的机制化长跑

### 5.1 编年总图

```
2025-05-01 task 工具首现（subagent 雏形，先有工具后有概念）
2025-06-12 a454ba8895 "subagent"（首提交：session/system.ts +75 行，task.ts +39 行）
2025-06-17 57b3051024 fix agent getting caught in summary loop  ← 子代理陷入循环
2025-06-19 97c4815444 fix task agent performance issues
2025-07-19 8f64c4b312 子代理裁剪 todo 工具（§2.3）
2025-07-20 #1149 防止 session_complete hook 在子代理会话误触发
2025-08-15 07dbc30c63 TUI 导航父子会话 / 08-16 #1981 切换保留缓存
2025-09-13 #2569 command 内 subagent + subtask 标志
2025-11-03 #3772 task 展示 [subagent type]
2025-11-19 #4204 task 工具 resume 能力
2025-12-15 b0f77da56c primary agents 与 subagents 配置分离
───────────────────────────────────────────────────── V2 分界
2026-04-08 #21017 task tool from agent services      ← ① registry 化
2026-04-29 #小模型化（§1.4）                          ← ② 成本分层
2026-05-09 #26597 deny 继承 / 06-10 #31696 self perms ← ③ 权限语义
2026-06-06 74a269af7d v2 background task tool         ← ④ 后台化
2026-06-20 #33142 honor configured agent step limits  ← ⑤ 步数限额
2026-07-12 #36530 后台子代理完成通知
2026-09-03 #47081 command subagents in background     ← ⑤ 命令亦后台
2026-09-09 #47455 后台子代理链接到其会话
```

### 5.2 task 工具的"三化"

task 工具从 39 行的玩具（a454ba8895）到 V2 编排原语，走了四次蜕皮：

| 化 | 事件 | 解决的痛点 |
|---|---|---|
| 小模型化 | 2026-04-29 只给 explore 配小模型 | 子代理烧钱、慢 |
| registry 化 | #21017 从 agent services 构建工具；06-06 与统一工具架构对齐（#31168） | task 是特例还是工具？统一注册 |
| 后台化 | #74a269af7d background task tool；#47081 命令后台 | 父会话被阻塞（#5887 异步委派 👍99 的回应） |
| 权限语义化 | #26597/#31696/c26f6f95f7 | 继承不可逃逸 + 自治不被吞掉 |

每一步都不是重写，是给同一个工具补一层机制——与架构线"推倒重写"的勇气模式
（README 阅读线索 2）恰成对照：**多智能体线是缝补史，不是重写史**。（对照为分析级观察。）

### 5.3 V2 机制化与未来

V2 把 subagent 语义固化为三件套——派生流（subagent.ts execute）、后台作业
（subagent-job.ts）、完成检测与结果回流（subagent-completion.ts）——细节全部留给
6_modules/12_subagent，此处只指出历史衔接：这三件套分别对应本篇三条痛史线的终点
（委派语义 §1、阻塞痛点 §2 之 5887、权限事故 §4.3）。

未完的部分：#5887（真异步后台子代理 👍99）的完全体、#8751（agents/skills 热重载 👍114）、
以及 §1.5 的 Agent Teams——多智能体线与形态线在同一处汇合，等待下一个时期。

---

## 6. 收官：第五线与架构线的交织

### 6.1 交织表：谁倒逼谁

| 时期 | 架构事件（1-4 篇） | agent 技术事件（本篇） | 倒逼方向 |
|---|---|---|---|
| Go 期 | TS 重写决策 | 单 loop、bash/edit 出生 | **agent→架构**：TS 生态的 LLM 工具链是重写动机之一 |
| 雏形期 | 工具目录成形 | 七工具补全、subagent 首提交 | **agent→架构**：工具面膨胀逼出 tool/ 目录与权限系统 |
| V1 期 | 单包膨胀 437 文件 | 7 agent 硬编码、edit 27 修、plan 钟摆 | **架构→agent**：巨石里改不动 agent 语义（permission rework 修在 652 行测试上仍难伸展） |
| V1→V2 | Effect 迁移动员 | #6480 mid-turn、#6319 rework | **agent→架构**：step 级检查/求值需要 Effect 的确定性失败——V2 为 agent 语义铺路 |
| V2 期 | schema/core、事件溯源 | deny 继承、后台化、registry 化、插件化 prompt | **架构→agent**：事件溯源让 subagent-job/completion 可审计可恢复 |

（方向判定为分析级，依据是事件先后与依赖关系。）

### 6.2 痛点→机制对照补充（衔接 appendix §7）

| 本篇痛点 | 沉淀机制 | 笔记落点 |
|---|---|---|
| #152/#6152 窗口焦虑 | usage 可见 + compaction 三路触发 | 6_modules/7_compaction |
| #5480/#11995 上下文偷吃 | 工具输出限流 + minimize prompt | 6_modules/6_context §2 |
| #23595/#5550 缓存失效 | durable instruction 三表 / cache breakpoints | 6_modules/6_context §4 |
| plan 钟摆（§1.3） | plan enter/exit 工具 + runtime flags | 6_modules/8_agent §4 |
| #26597 绕过 | subagent-permissions 继承 + 回归测试 | 6_modules/11_permission |
| #5887 阻塞 | subagent-job / background task | 6_modules/12_subagent §3 |
| edit 27 修 | Tool.defineEffect + 模型分诊（#34558 gate by model） | 6_modules/9_tool |
| yolo 呼声 | yolo mode（#33279） | 6_modules/11_permission |
| #12661 Teams | 未定论：task+后台 vs Team 原语 | 本文 §1.5/§5.3 |

### 6.3 一句收卷

架构线讲的是**这个项目三次推倒自己的房子**；本篇讲的是**住户在旧房子里学会的每一课
——工具要裁着给、上下文要省着花、权限要继承着管、子代理要放出去又收得回**。
V2 之所以值得三次豪赌，正因为这些课程只有在新地基（Effect/事件溯源/schema）上
才能从"补丁"变成"机制"。Agent 技术线与架构线在 2026 年合流：**机制化的 agent，
是 V2 重建真正的客户。**

---

## 附：本篇考古命令（可复现）

```bash
# 七 agent 清单（V1 峰值）
git show c8ecd640:packages/opencode/src/agent/agent.ts | grep -nE 'name: "|mode: "'
# edit 27 连修
git log --oneline --all -i --grep="edit tool" | wc -l        # 27
# plan 钟摆
git log --format="%ad %h %s" --date=short -i --grep="plan mode"
# 权限编年
git log --format="%ad %h %s" --date=short -i --grep="permission" | tail -40
# #26597 事故修复与测试
git show b8ca71d309 --stat
# task 三化
git log --format="%ad %h %s" --date=short -i --grep="task tool\|subagent"
# 上下文事故线
git log --format="%ad %h %s" --date=short -i --grep="compact" | tail -15
git show 7a3ff5b98f --stat        # mid-turn 溢出检查（#6480）
```
