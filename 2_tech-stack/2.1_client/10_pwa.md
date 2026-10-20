# PWA（Progressive Web App）

## 是什么

让网站获得**类原生应用能力**的 Web 标准方案组合：Service Worker（离线缓存、后台能力）+ Web App Manifest（安装到桌面/主屏）+ HTTPS。不是框架，是一套标准。

## 业界类比

- "介于网页与原生 App 之间的形态"
- 安装后无浏览器外壳、可离线启动

## 在 OpenCode 中的角色

- **Web 端（`@opencode/app`）的增强能力**：支持安装为独立应用、离线壳体验
- 由 Vite 插件体系接入（构建期生成 manifest 与 SW 产物）

## 与其他技术的关系

- **Vite**：PWA 插件在构建期注入（见 [vite.md](4_vite.md)）
- **桌面端**：完整桌面体验走 Electron；PWA 是"不想装 Electron"的轻量替代路径，二者共用同一 Web 应用

## 相关源码

- `packages/app/`（PWA 配置）
