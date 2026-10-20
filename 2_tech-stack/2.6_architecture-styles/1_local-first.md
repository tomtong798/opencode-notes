# 本地优先（Local-First）

## 是什么

软件形态哲学：**数据与计算默认在本机**，云只是可选增强（分享/同步/备份）。核心功能断网可用，用户数据先落在自己的磁盘。

## 业界类比

- 对比 SaaS/云优先：数据先在厂商库里、断网不可用
- Ink & Switch 的 local-first 宣言（CRDT 协同一系）；opencode 是"单机为主、云为增值"的简化版
- 传统桌面软件（Office 时代）+ 现代分发/更新的合体

## 在 OpenCode 中的角色

- **opencode 的根本形态**：
  - 全部会话/配置/凭据存本地 SQLite 与文件（XDG 目录）
  - agent 引擎、LLM 调用、工具执行全在本机进程（唯一外部依赖是 LLM API 本身）
  - 断网除模型调用外一切可玩：历史浏览、配置、TUI
- 云侧仅增值：会话分享（Durable Objects）、stats 匿名统计、更新服务
- 分发形态印证：curl/npm/brew 装到本机，而非注册 SaaS 账号

## 与其他技术的关系

- **守护进程架构**：本地"服务端"的载体（见 [daemon-client.md](2_daemon-client.md)）
- **SQLite**：本地数据的物理形态（见 [../2.3_data/sqlite.md](../2.3_data/1_sqlite.md)）
- **Cloudflare 侧**：全部是"可选云能力"（见 [../2.4_infrastructure/](../2.4_infrastructure/)）

## 相关源码

- 全仓拓扑；`packages/core/src/config/`（本地多源配置）、`~/.local/share/opencode`（数据目录约定）
