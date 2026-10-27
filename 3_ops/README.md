# 运维与排障笔记（Ops Notes）

> 回答"**装在哪、日志在哪、怎么排障**"——部署后的磁盘布局与运行时位置。
> 全部路径从代码实测派生：v2.0.30，`dev` 分支（2026-10-16）。

## 索引

| 篇目 | 主题 |
|---|---|
| [log-locations.md](1_log-locations.md) | 两套日志体系：后端 fileLogger（单文件+原地压缩）与桌面 Electron（时间戳目录+7 天清理） |
| [deployment-layout.md](2_deployment-layout.md) | XDG 全局布局、按 channel 分库、项目 `.opencode/` 发现规则、桌面 userData、环境变量总表 |
| [deb-package-name.md](3_deb-package-name.md) | Debian 打包：DEBIAN/control 的 Package 字段如何决定 apt remove 的名字（迁自 gitee 旧库，通用打包知识） |
| [bun-run-dev-flow.md](4_bun-run-dev-flow.md) | `bun run dev` 开发启动全流程（迁自 gitee 旧库；基于个人开发环境，非上游 dev 基线） |

## 姊妹目录

- [`../1_architecture/`](../1_architecture/README.md) — 系统怎么组织/怎么运转
- [`../2_tech-stack/`](../2_tech-stack/README.md) — 技术是什么
