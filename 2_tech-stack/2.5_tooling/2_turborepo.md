# Turborepo

## 是什么

**monorepo 任务编排器**：按包依赖图调度任务（build/typecheck/test），带内容寻址缓存（本地/远程）与并发控制——未变的任务直接命中缓存跳过。

## 业界类比

- "给 monorepo 加增量构建 + 缓存层"（Bazel/Nx 的轻量替代）
- turbo = 任务图的 make，缓存 = 内容哈希

## 在 OpenCode 中的角色

- 根命令：`bun turbo typecheck --concurrency=3`（typecheck 任务的统一入口；lint 是 oxlint + ast-grep 直跑不经 turbo）
- **transit 任务**（仓库特色用法）：turbo.json 定义 transit 任务把 workspace 依赖折叠进测试的哈希输入——测试对依赖包源码变化敏感，**但不需要先实际构建依赖**（大部分包是源码直跑，无 build 产物）
- 显式 `dependsOn` 链：cli→core、sdk→core、server→core 等

## 与其他技术的关系

- **Bun workspaces**：包拓扑来源（见 [bun-workspaces.md](1_bun-workspaces.md)）
- **GitHub Actions**：CI 内由 turbo 编排 typecheck/test
- **Bun test**：test 任务的实际执行器

## 相关源码

- `turbo.json`
