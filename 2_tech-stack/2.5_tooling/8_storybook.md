# Storybook（10.x）

## 是什么

**组件工作台**：把 UI 组件从应用里 isolated 出来，配"故事"（story = 组件的某个状态组合）交互式开发、文档化与视觉回归。

## 业界类比

- "组件的美术馆/标本馆"
- 组件驱动开发（CDD）的标准工具

## 在 OpenCode 中的角色

两种形态：

1. **Web 组件**：`packages/storybook`（Storybook 10 + storybook-solidjs-vite），覆盖 `@opencode/ui` / `@opencode/session-ui` 组件；`bun run dev:storybook` 启动
2. **TUI 故事**（仓内自建、非 Storybook 产品）：`packages/tui/src/feature-plugins/system/storybook/`——fixture 驱动渲染**真实生产组件**（AGENTS.md 规则），故事键位暴露状态维度、副作用本地化；`OPENCODE_STORY=<id> bun run dev:live` 运行，并要求窄/宽两种终端尺寸都验证

## 与其他技术的关系

- **Playwright**：组件测试在 story 上跑（见 [playwright.md](7_playwright.md)）
- **SolidJS / Vite**：storybook-solidjs-vite 渲染器
- **OpenTUI**：TUI 故事体系渲染到终端

## 相关源码

- `packages/storybook/`、`packages/tui/src/feature-plugins/system/storybook/`、`.opencode/skills/component-tests`
