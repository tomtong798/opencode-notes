# Tailwind CSS（4.x）

## 是什么

原子化（utility-first）CSS 框架：不写传统样式表，直接在标记上组合 `flex p-4 text-sm` 这类单一职责工具类，由扫描器生成最小 CSS。v4 用 Rust 重写引擎（Oxide），配置移入 CSS 本身（`@theme`），速度大幅提升。

## 业界类比

- "把 CSS 变成可组合的乐高积木"
- 与传统组件样式（BEM/SCSS）相反：放弃语义化 class，换极致一致性
- v4 是"零配置 + CSS-first 配置"的分水岭版本

## 在 OpenCode 中的角色

- **Web/桌面端样式方案**（`@opencode/app`、`@opencode/ui`、console/stats 站点）
- 与 **Kobalte** 的"无样式"互补：Kobalte 管行为，Tailwind 管外观，合成本仓库自研组件库 `@opencode/ui`
- 内置 Shiki 高亮、KaTeX、图标体系与 Tailwind 共同构成 UI 包的视觉底座

## 与其他技术的关系

- **Kobalte**：headless（行为）+ Tailwind（样式）= 完整组件（见 [kobalte.md](6_kobalte.md)）
- **SolidJS**：Tailwind 与框架无关，class 写在 SolidJS 组件标记上
- **TUI 不用 Tailwind**——终端 UI 走 OpenTUI 的语义化主题 token 体系（两套"主题"系统并存，勿混淆）

## 相关源码

- `packages/ui/`（`./styles` 导出）、`packages/app/`
