# OpenCode 端到端时间线专卷（Timeline Notes）

> 对象：`/home/donghe/github/opencode`（v2.0.30，`dev` 分支）。
> 定位：**一条端到端时间线**——从双击桌面图标，到后端服务就绪、界面可用、用户提问、模型响应（含 TUI/CLI 终端路径）。按时间顺序组织，每篇都是**代码级走读**（带 `文件:行号`，可对照源码逐行分析）。
> 分工：Electron 框架本体 → [`../2_tech-stack/2.1_client/8_electron.md`](../2_tech-stack/2.1_client/8_electron.md)；桌面进程架构细节 → [`../4_desktop/`](../4_desktop/README.md)；全局架构 → [`../1_architecture/`](../1_architecture/README.md)。

---

## 时间线总图

```
双击图标
═══════════════════════════════════════════════════════════════════════
[1] 桌面壳启动          1_desktop-boot.md
    主进程环境准备 → 窗口先行 → 并行探测/拉起后端服务 →
    MessagePort 交接 → renderer 直连服务
═══════════════════════════════════════════════════════════════════════
[2] 后端服务初始化      2_server-init.md
    opencode serve --service → 端口绑定 → 全局服务树（25 节点）
    → Database(WAL+迁移) → HttpApi 装配 → status.ready → /api/info 200
    （与 [1] 并行发生——桌面端 ensure() 拉起的就是这条链）
═══════════════════════════════════════════════════════════════════════
[3] Location 惰性编译   3_location-lazy.md
    首个携带目录的请求 → LocationServiceMap.get(ref) →
    Instance.layer 编译 53 个 Location 服务（Agent/Tool/Permission/
    SessionRunner 依赖全就绪）→ 缓存（永不过期）
    （"服务秒起、首请求稍慢"的原因）
═══════════════════════════════════════════════════════════════════════
[4] 前端数据同步        4_renderer-boot.md
    SSE 连接（server.connected 握手）→ 全局 bootstrap（config/
    path/projects）→ 目录 bootstrap（12 类资源并行拉取）→ store 就绪
═══════════════════════════════════════════════════════════════════════
[5] 提问执行链          5_prompt-chain.md
    回车 → 乐观 UI → POST session.prompt → Inbox 准入（durable 事件）
    → wake/drain → Step：llm.stream 流式 → 工具并行 fiber →
    事件回流 Bus → SSE → store 投影 → 界面渲染
═══════════════════════════════════════════════════════════════════════
[5'] 终端提问链（姊妹篇） 6_tui-cli-prompt-chain.md
    TUI：footer 回车 → 串行队列（active 期间直发 durable 准入）→
    switchAgent/Model → session.prompt → SSE apply reducer →
    session.wait idle fence；CLI：单发 steer → 事件循环 +
    权限自动拒绝 → session.wait + reconcile 投影对账
```

## 阅读方式

- **顺读**：按 1→5 就是完整的"从开机到对话"叙事
- **跳读**：每篇开头有前置依赖声明；`[2]`/`[3]` 也适用于 CLI/TUI 场景（同一 server，不依赖桌面端）
- 全部行号基于 v2.0.30（2026-10-16 测量），代码演进后请以符号名检索为准

## 索引

| 篇 | 主题 |
|---|---|
| [1_desktop-boot.md](1_desktop-boot.md) | Electron 壳启动：T1→T11 时序 |
| [2_server-init.md](2_server-init.md) | 后端服务初始化：入口→数据库→服务树→就绪门 |
| [3_location-lazy.md](3_location-lazy.md) | Location 惰性编译：首请求编译 53 服务 |
| [4_renderer-boot.md](4_renderer-boot.md) | 前端数据同步：SSE/双阶段 bootstrap/store |
| [5_prompt-chain.md](5_prompt-chain.md) | 提问执行链：回车到流式渲染全链路 |
| [6_tui-cli-prompt-chain.md](6_tui-cli-prompt-chain.md) | 终端提问链：TUI 队列与 CLI 单发（含三端对比、旧笔记漂移清单） |

## 端到端时序图（puml/）

PlantUML 源文件 + 渲染产物（png/svg），内容按 v2.0.30 现码绘制（旧库 2026-08-03 的 puml 因 `prompt_async` 删除、TUI 队列语义重写已过时，未搬运）：

| 图 | 覆盖 |
|---|---|
| [0_e2e-overview](puml/0_e2e-overview.puml) | 全流程俯瞰：双击图标→启动→同步→提问→渲染（串起 1-6 篇） |
| [1_desktop-input-to-response](puml/1_desktop-input-to-response.puml) | Desktop/Web：乐观更新 + 幂等准入 + 7 跳事件回程 |
| [2_tui-input-to-response](puml/2_tui-input-to-response.puml) | TUI：串行队列双分支 + `session.wait` idle fence + fragments 对账 |
| [3_cli-non-interactive-flow](puml/3_cli-non-interactive-flow.puml) | CLI：steer 单发 + 权限自动拒绝 + reconcile 投影对账 |

重渲染：`java -Djava.awt.headless=true -jar plantuml.jar -tpng -tsvg puml/*.puml`
