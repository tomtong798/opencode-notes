# OpenCode 发展史专卷（History）

> 对象：`/home/donghe/github/opencode`（remote: anomalyco/opencode，22 万 star）。
> 定位：**纵向时间轴**——从建库到现在的演进史，每个时期的痛点与解法。与 [../survey/](../survey/README.md)（横向对比）相对；与 [../1_architecture/6_v1-to-v2-evolution.md](../1_architecture/6_v1-to-v2-evolution.md)（代码内证据的 V1/V2 对照）互补，本卷以 **git 全史 + GitHub issue/PR 数据**为源。
> 数据基线：2026-10-22 采集——本地 git 全史 20,483 commits（2025-03-21 起）；建库初期 7 周 issue/PR 全量 1,000 条（4-30~06-19，去重后 723 issues + 277 PRs）；全期 31,134 issues 的反应数榜单（search API 抽样）。

---

## 一、四段式演进总图

```
2025-03-21 ──── 2025-05-30 ──── 2025-07 ────── 2026-03 ────── 2026-07-20 ──── 2026-10 →
   │               │               │               │               │              │
   【1】Go 原型      【2】TS 重写      【3】V1 膨胀期     【4】V2 重建        【5】V2 收尾
   "termai"         8 周完成        opencode 单包    Effect 迁移计划     V1 主入口删除
   8 commits       go.mod 删除      膨胀到 437 文件   schema/core 诞生   插件宿主定型
                                   渲染性能/权限/    OpenTUI 自研       2026-10 仍
                                   模型碎片三线作战  事件溯源铺开        663 commits/半月
```

月提交量（演进节奏的最诚实指标）：

```
2025-03 ████ 8            2025-11 █████████████████████ 1049
2025-04 ██████ 152        2025-12 ██████████████████████████████████ 1877
2025-05 ████████ 301      2026-01 ██████████████████████████████████████ 2083 (峰值)
2025-06 █████████ 654     2026-02 ████████████████ 1165
2025-07 █████████ 650     2026-03 █████████████ 963
2025-08 █████████ 614     2026-04 ████████████████ 1328
2025-09 ████████ 518     2026-05 █████████████████ 1551
2025-10 █████████ 709     ...
                         2026-08~10 维持 1500/1100/663 高位
```

## 二、篇目

| # | 篇 | 时期 | 核心痛点 |
|---|---|---|---|
| 1 | [1_go-prototype.md](1_go-prototype.md) | 2025-03~05 | 语言选型错了，8 周弃船 |
| 2 | [2_ts-rewrite-and-early-days.md](2_ts-rewrite-and-early-days.md) | 2025-05~07 | TS 重写 8 周冲刺 + 雏形期三家牌桌（权限/渲染/模型） |
| 3 | [3_v1-growth-pains.md](3_v1-growth-pains.md) | 2025-07~2026-03 | 单包膨胀 117→437 文件、修不动的 TUI、Effect 化两难 |
| 4 | [4_v2-rebuild.md](4_v2-rebuild.md) | 2026-03~10 | 三次豪赌：Effect 全家桶 / 自研 OpenTUI / 事件溯源推倒重来 |
| 5 | [5_agent-capability-evolution.md](5_agent-capability-evolution.md) | 贯穿全程 | **agent 技术本体**：工具/上下文/权限/多智能体的痛史（与架构线交织的第五线） |
| 6 | [6_layernode-story.md](6_layernode-story.md) | 2026-06~08 | **单抽象深挖**：LayerNode 诞生记——为什么在 Effect 之上再造装配图（PR 编年 + "没有它会怎样"推演） |

## 三、贯穿全期的三条线（阅读线索）

1. **痛点 → 机制的对偶**：全期高票 issue 几乎全部沉淀为今天代码库的某个机制（[数据附录](appendix-data.md)的对照表）——社区的"困难清单"与 [../6_modules/](../6_modules/README.md) 的"机制清单"互为镜像。
2. **推倒重写的勇气模式**：Go→TS（8 周丢掉）、V1→V2（437 文件包退役）、TUI 两次重写——这个项目在"修补 vs 重写"之间反复选择后者。
3. **复杂度 = 生存成本**：你在 [../7_nonfunctional/README.md](../7_nonfunctional/README.md) 读到的每条防御机制，几乎都能在 [数据附录](appendix-data.md) 里找到对应的 issue 痛点。

## 四、数据附录

[appendix-data.md](appendix-data.md)——本卷全部原始证据：月提交量表、包诞生时间线、V1 fix 词频、热门 issue 榜、关键 commit 编年。

## 约定

- 可信度标注：git 考古与 issue 数据为**实证**；动机推断（如"为什么放弃 Go"）标注为**分析**；
- 全卷行号/commit 可复现：`git log --format="%ad %h %s" --date=short <条件>`；
- issue 数据采集于 2026-10-22，GitHub API 有限流，抽样方法记录在附录。
