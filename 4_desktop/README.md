# OpenCode 桌面端专卷（Desktop Notes）

> 对象：`/home/donghe/github/opencode`（v2.0.30，`dev` 分支）的 `packages/desktop`。
> 定位：**Electron 壳层视角**——进程拓扑、通信通道、安全配置、sidecar 托管、桌面自有存储。
> 启动时间线（图标→服务→可交互→提问→响应）已迁至 [`../5_timeline/`](../5_timeline/README.md) 按时序专述。
> 分工：Electron 框架本体见 [`../2_tech-stack/2.1_client/8_electron.md`](../2_tech-stack/2.1_client/8_electron.md)；全局架构见 [`../1_architecture/`](../1_architecture/README.md)。

---

## 阅读顺序

1. [1_multi-process.md](1_multi-process.md) — 多进程架构：拓扑、双通信通道（MessagePort RPC + HTTP）、安全配置、sidecar 形态与桌面自有存储
2. [`../5_timeline/1_desktop-boot.md`](../5_timeline/1_desktop-boot.md) — 桌面壳启动时序（T1→T11，本卷的前置续篇）

## TODO（按需扩卷）

| 篇目 | 一句话 |
|---|---|
| 2_extension-host.md | 桌面扩展宿主：安装/启停/嵌入/资产服务（`src/main/extension/`） |
| 3_storage.md | 桌面存储细化：drafts.sqlite 三表、write-behind、blob GC、窗口状态 |
| 4_updater.md | 自动更新全流程：electron-updater + services/update 按通道分流 |
