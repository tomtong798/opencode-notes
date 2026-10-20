# OpenCode 学习笔记（opencode-notes）

> 对象：`/home/donghe/github/opencode`（v2.0.30，`dev` 分支）。
> 全部结论基于现码实测，行号测量于 2026-10-16；代码演进后以符号名检索为准。

## 目录

| 卷 | 定位 |
|---|---|
| [1_architecture/](1_architecture/README.md) | 宏观全局：分层/依赖/模式/演进（含 TODO 引擎深挖） |
| [2_tech-stack/](2_tech-stack/README.md) | 技术是什么：一项技术一篇（53 篇，含 Effect 家族 6 篇） |
| [3_ops/](3_ops/README.md) | 运维排障：日志位置、部署目录 |
| [4_desktop/](4_desktop/README.md) | 桌面端专卷：Electron 壳层（进程/通道/存储） |
| [5_timeline/](5_timeline/README.md) | 端到端时间线专卷：图标→服务→Location→前端→提问→终端提问（代码级走读） |
| [6_modules/](6_modules/README.md) | 模块详解卷：子系统内部结构（Effect 基建/数据层/Session/工具等 17 篇规划） |
| [survey/](survey/README.md) | 横向调研：跨仓库对比、行业趋势（含 AgentScope 系列 4 篇） |

## 约定

- 目录与文件均带阅读序号；`x.y` 文件号表示从属关系（如 `3.1_effect-schema.md` 隶属 `3_effect.md`）
- 每篇统一结构：一句话概括 → 结构图 → 关键文件（带行号）→ 走读 → 设计取舍 → 相关篇互链
- 权威优先级：`AGENTS.md` > `specs/v2/` > 各包 AGENTS.md > 源码 > 本笔记
