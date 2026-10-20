# SolidJS

## 是什么

声明式响应式 UI 框架，React 的替代品。核心特征是**细粒度响应式（fine-grained reactivity）**：

- 依赖追踪精确到**信号（signal）**级别，而非组件级别
- 组件函数**只运行一次**（没有 re-render）
- 没有**虚拟 DOM**，直接精确更新 DOM 节点（或任意渲染目标）

## 业界类比

- React："组件重跑 + vdom diff"；SolidJS："组件只跑一次 + 精确订阅更新"
- Vue 的响应式思路 + React 的 JSX/组件心智 + 更激进的性能
- Signals 模型的代表实现（后续 Vue/Preact/Angular 都引入了 signals）

## 在 OpenCode 中的角色

- **四端共享的"组件语言"**：
  - TUI：经 `@opentui/solid` 渲染到终端（见 [opentui.md](3_opentui.md)）
  - Web：原生渲染到浏览器 DOM（`@opencode/app`）
  - 桌面：Electron 装载同一套 Web 应用
- **状态管理不用 Redux/Pinia 类专用库**，用 SolidJS 原生 store（`createStore`/`createSignal`/`createResource`）：
  - `@opencode/2.1_client/solid` 的 `createData()` 把服务端 SSE 事件流投影为响应式 store（会话/消息/模型/权限），TUI 与 Web 共用
  - TUI 的 `context/` 层全部基于 `createSimpleContext()` 模式包装 store
- 路由：Web 端用 solid-router；TUI 端自建三路由 store（home/session/plugin）

## 与其他技术的关系

- **OpenTUI**：`@opentui/solid` 是 SolidJS 的自定义渲染器——同一套组件写法，渲染目标从 DOM 换成终端字符单元格
- **Kobalte**：SolidJS 生态的 headless 组件库（行为层）
- **TanStack Solid Query**：TanStack Query 的 SolidJS 移植
- **Tailwind CSS**：样式层，与组件模型正交
- **Vite**：`vite-plugin-solid` 提供 JSX 转译（`dev/vite.ts` 中 `ensureSolidTransformPlugin()`）

## 相关源码

- TUI 应用壳：`packages/tui/src/app.tsx`（25+ provider 嵌套）
- 共享数据层：`packages/client/src/solid/data.ts`、`connection.ts`
- Web 应用：`packages/app/src/`
- 组件库：`packages/ui/`、`packages/session-ui/`
