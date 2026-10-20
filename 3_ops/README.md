# 运维与排障笔记（Ops Notes）

> 回答"**装在哪、日志在哪、怎么排障**"——部署后的磁盘布局与运行时位置。
> 全部路径从代码实测派生：v2.0.30，`dev` 分支（2026-10-16）。

## 索引

| 篇目 | 主题 |
|---|---|
| [log-locations.md](1_log-locations.md) | 两套日志体系：后端 fileLogger（单文件+原地压缩）与桌面 Electron（时间戳目录+7 天清理） |
| [deployment-layout.md](2_deployment-layout.md) | XDG 全局布局、按 channel 分库、项目 `.opencode/` 发现规则、桌面 userData、环境变量总表 |

## 姊妹目录

- [`../1_architecture/`](../1_architecture/README.md) — 系统怎么组织/怎么运转
- [`../2_tech-stack/`](../2_tech-stack/README.md) — 技术是什么
