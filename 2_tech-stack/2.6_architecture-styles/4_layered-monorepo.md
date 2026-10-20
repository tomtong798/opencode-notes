# 严格分层 Monorepo（Layered Monorepo）

## 是什么

**依赖方向作为仓库级铁律**的 monorepo 组织方式：包之间只允许单向依赖，方向即架构。

## 业界类比

- 洋葱/六边形架构的"包粒度版"：领域在内、传输在外
- 对比自由 monorepo：依赖图被人为约束成 DAG 且方向固定

## 在 OpenCode 中的角色

**依赖铁律**（根 AGENTS.md 明文）：

```
Schema → Core → Protocol → Server
                Protocol → Client（Client 还可用 Schema，但永不 Core/Server）
SDK = Client + Core + Server 的组合层
```

- **意义**：
  - Client 可独立发布（不拖服务端代码）——外部应用嵌 opencode 的入口
  - 契约（Schema/Protocol）可被所有层安全复用
  - 防止领域逻辑反向泄漏进传输层
- 分层之外的辅助约束：`.opencode/skills` 按测试形态（unit/tui/app-e2e/component）放约定；`specs/v2/` 记录跨模块契约；根 AGENTS.md 是贡献红线权威
- CI 强制：typecheck 沿 turbo 依赖图跑，越界依赖会破坏构建

## 与其他技术的关系

- **契约先行**：分层的执行配对（见 [contract-first.md](3_contract-first.md)）
- **Bun workspaces / catalog**：依赖拓扑的物理载体（见 [../2.5_tooling/bun-workspaces.md](../2.5_tooling/1_bun-workspaces.md)）
- **包版图**：load-bearing（core/tui/cli/app/ai…）vs peripheral（营销站/内部运营）vs 空壳，详见 `tmp/opencode-codebase-overview.md` 第 5 节

## 相关源码

- 根 `AGENTS.md`（第一条规则）、`specs/v2/README.md`（Authority 表）、`packages/schema/AGENTS.md`
