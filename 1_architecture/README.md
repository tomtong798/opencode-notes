# OpenCode 架构笔记（Architecture Notes）

> 对象：`/home/donghe/github/opencode`（v2.0.30，分支 `dev`）。
> 本目录回答"**系统怎么组织、怎么运转**"——一篇一机制；文件序号 = 推荐阅读顺序。
> "这是什么技术"类问题见姊妹目录 [`../2_tech-stack/`](../2_tech-stack/README.md)（50 篇，一项技术一篇）。

---

## 阅读路径

### 入门三篇（按序）

1. [1_overview.md](1_overview.md) — 全局导览：分层、各子系统、请求生命周期（先建心智地图）
2. [2_package-map.md](2_package-map.md) — 包版图与依赖实测（知道东西都在哪）
3. [3_metrics.md](3_metrics.md) — 规模数据：LOC 分布、测试分布（知道重心在哪）

### 进阶三篇（按需）

4. [4_build-and-packaging.md](4_build-and-packaging.md) — 源码如何变成产物：CLI 双路径/Electron/Web/发布流水线
5. [5_patterns.md](5_patterns.md) — Effect DI 根模式 + GoF 清单 + 仓内惯用法
6. [6_v1-to-v2-evolution.md](6_v1-to-v2-evolution.md) — 架构演进：时间线 + 三个兼容面现状

### TODO（用到再写）

| 篇目 | 一句话 |
|---|---|
| 7_session-engine.md | 引擎内幕：inbox 准入 → drain → step → 投影 全链路源码走读 |
| 8_event-flow.md | 事件的一生：Bus → projector → SSE → createData → UI 渲染 |
| 9_tool-system.md | 工具子系统：注册/权限门控/并行执行/bounding + 16 内置工具 |
| 10_extension-points.md | 扩展点地图：加工具/插件/GUI 特性/API 各改哪里 |
| 11_data-model.md | 数据模型：表结构 + ~40 种 session 事件类型清单 |

---

## 篇目规范

每篇统一六段结构：

1. **一句话概括**（这个机制解决什么问题）
2. **结构图**（ASCII 为主，Mermaid 可选）
3. **关键文件与符号**（带路径，可直接跳源码）
4. **走读/流程**（按执行顺序讲一遍）
5. **设计取舍**（为什么这样做、约束来自哪里——引用 `AGENTS.md` / `specs/v2/` 红线）
6. **相关篇互链**

**权威优先级**：本目录是学习笔记，代码与官方文档永远是权威——
`AGENTS.md`（贡献红线）> `specs/v2/`（跨模块契约）> 各包内 `AGENTS.md` > 源码本身 > 本目录。

> 目录膨胀到 20+ 篇前保持扁平；届时再按主题重分。

## 姊妹目录

- [`../2_tech-stack/`](../2_tech-stack/README.md) — "这是什么技术"（50 篇，一项技术一篇）
- [`../3_ops/`](../3_ops/README.md) — 运维排障：日志位置、部署目录清单
- [`../4_desktop/`](../4_desktop/README.md) — 桌面端专卷：Electron 使用与启动全流程
