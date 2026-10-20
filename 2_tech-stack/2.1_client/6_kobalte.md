# Kobalte

## 是什么

SolidJS 生态的 **headless（无头）组件库**：提供 Dialog、Select、Tooltip、Combobox 等组件的完整**行为与无障碍语义**（ARIA 属性、焦点管理、键盘导航），但**不带任何样式**——外观完全交给使用方。

## 业界类比

- SolidJS 界的 Radix UI（React 生态 headless 标杆）
- "行为我包，长相你画"

## 在 OpenCode 中的角色

- 与 Tailwind CSS 4 配对，构成自研组件库 `@opencode/ui` 的行为层：Kobalte 管交互/无障碍，Tailwind 管视觉
- Web 端（`@opencode/app`）、console/stats 站点同样基于这对组合

## 与其他技术的关系

- **SolidJS**：Kobalte 组件即 SolidJS 组件，响应式由 signals 驱动
- **Tailwind CSS**：天然互补（headless ↔ utility-first），见 [tailwind-css.md](5_tailwind-css.md)
- **dnd-kit（solid）**：拖拽能力与 Kobalte 组件组合使用（`@opencode/ui` 中）

## 相关源码

- `packages/ui/`（按组件细粒度导出：`./button`、`./dialog`、`./text-input`…）
