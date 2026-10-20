# OpenTUI

## 是什么

opencode 生态自研的**终端渲染框架**，包含几个包：

- `@opentui/core`：渲染器——把组件树绘制到终端字符单元格，处理布局（yoga flex）、键盘/鼠标输入、自适应尺寸，目标 60FPS
- `@opentui/solid`：**SolidJS 自定义渲染器适配层**——把 SolidJS 组件树的创建/更新重定向到终端而非 DOM
- `@opentui/keymap`：键映射系统（层/命令/绑定/leader 键/模式）

内嵌 **kitty 键盘协议**支持：终端键盘输入增强协议，能区分 Esc 与 Alt 组合、识别修饰键堆叠，是完整键位支持的基础，由渲染器协商启用。

## 业界类比

- `@opentui/solid` 之于终端 ≈ ReactDOM 之于浏览器 DOM
- Ink（React 写 CLI）的 SolidJS + 高性能对应物；目标是"终端里的原生 GUI"

## 在 OpenCode 中的角色

- TUI（`packages/tui`）的渲染底座：60FPS 渲染、kitty 协议、鼠标支持、外部输出直通
- 主题体系基于**语义化 token**（`packages/theme` + TUI 硬规则：禁裸 `theme.hue`、禁挪用无关 token）
- 测试两种形态：OpenTUI 渲染器测试（`.opencode/skills/tui-tests`）+ fixture 故事（`feature-plugins/system/storybook/`，`OPENCODE_STORY=<id> bun run dev:live` 运行）

## 与其他技术的关系

- **SolidJS**：组件模型来自 SolidJS，OpenTUI 提供渲染目标
- **Bun**：TUI 直接由 Bun 运行 TS 源码；开发态经 `packages/cli/dev/vite.ts` 做 JSX 转译并支持热重载（Vite 通道）
- **@opencode/merman / latex**：TUI 插件，渲染 Mermaid 图与 LaTeX
- **keymap**：TUI 的键位系统（`context/keymap.tsx` 包装为 context）

## 相关源码

- TUI 入口：`packages/tui/src/app.tsx`（创建 CliRenderer）
- 键映射：`packages/tui/src/context/keymap.tsx`
- 主题：`packages/theme/`（`./tui` 当前、`./tui/v1` 旧版）、`packages/tui/src/theme/index.ts`
- 故事书：`packages/tui/src/feature-plugins/system/storybook/`
