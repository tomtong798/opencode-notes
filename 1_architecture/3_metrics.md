# 规模测量数据（Codebase Metrics）

> 一句话：**代码重心在 UI 不在引擎——UI 五包 ~317k LOC 是 core 的 4.7 倍；复杂度集中在三个热点。**
> 测量日期：2026-10-16，`dev` 分支（v2.0.30）。
> [返回索引](README.md) | [上一篇：包版图](2_package-map.md)

**本篇保持"可重复测量"**——头部附复测命令，代码演进后重跑即可刷新全部数据。

---

## 复测命令

```bash
# 主力包 LOC（ts+tsx，src 目录）
cd /home/donghe/github/opencode/packages
for d in <包名...>; do
  [ -d "$d/src" ] && echo "$d: $(find $d/src \( -name '*.ts' -o -name '*.tsx' \) | wc -l) files, \
  $(find $d/src \( -name '*.ts' -o -name '*.tsx' \) -exec cat {} + | wc -l) LOC"
done

# core 顶层目录文件数
cd core/src && for d in */; do echo "$d: $(find $d -name '*.ts' | wc -l)"; done | sort -t: -k2 -rn

# 测试文件分布
find packages \( -name '*.test.ts' -o -name '*.test.tsx' -o -name '*.spec.ts' \) \
  -not -path '*/node_modules/*' | sed 's|packages/||;s|/.*||' | sort | uniq -c | sort -rn
```

---

## 1. 主力包规模（src 目录，ts+tsx，按 LOC 降序）

| 包 | 文件数 | LOC | 角色 |
|---|---|---|---|
| **app** | 534 | **131,341** | Web 应用（最大包，也是桌面渲染器） |
| **core** | 425 | **68,098** | 引擎：session/agent/tool/provider/存储 |
| **tui** | 284 | **62,496** | 终端 UI（含独立 mini 第二 UI） |
| **gui-extensions** | 845 | **60,848** | 桌面/web GUI 扩展（文件数全仓第一） |
| **ui** | 216 | 36,421 | 共享组件库 |
| **ai** | 217 | 31,542 | LLM provider 抽象 |
| **session-ui** | 115 | 25,837 | 会话时间线组件 |
| **merman** | 88 | 21,542 | Mermaid 渲染（TUI 插件） |
| **client** | 32 | 16,968 | 生成客户端 + solid 数据层 |
| **cli** | 116 | 12,965 | 命令行入口 |
| **desktop** | 150 | 11,930 | Electron 壳 |
| **codemode** | 45 | 10,784 | 沙箱解释器 |
| protocol | 36 | 4,282 | HttpApi 定义 |
| util | 46 | 4,691 | 横切工具 |
| server | 52 | 4,098 | handlers + EventFeed |
| plugin | 63 | 3,369 | 插件 SDK |
| latex | 14 | 2,646 | LaTeX 渲染 |
| simulation | 13 | 2,182 | 会话模拟 |
| httpapi-codegen | 1 | 1,961 | 客户端生成器（单文件！） |
| theme | 10 | 1,592 | TUI 主题 |
| http-recorder | 14 | 1,596 | VCR 测试 |
| plugin-browser | 7 | 1,531 | 浏览器 RPC |
| sdk | 15 | 668 | 嵌入式组合层 |
| function | 3 | 573 | Cloudflare 函数 |

**UI 五包合计**（app+tui+gui-extensions+ui+session-ui）≈ **317k LOC = core 的 4.7 倍**。

---

## 2. core 内部分布（37 个顶层目录，按文件数）

### 第一梯队（引擎主体）

| 模块 | 文件数 | 备注 |
|---|---|---|
| database/ | 79 | 大头是 `migration.gen.ts`/`schema.gen.ts` 等**生成物** |
| plugin/ | 68 | hooks + 6 子域（command/provider/skill/system-prompt/vcs/websearch） |
| session/ | 49 | 引擎核心（见 §3） |
| config/ | 26 | discovery/normalize/variable/watch |
| github-copilot/ | 24 | Copilot 集成专档（量级惊人） |
| tool/ | 20 | 注册表 + runtime + 16 内置工具 |

### 第二梯队

shell/（7，tree-sitter 解析）、environment/（8）、filesystem/（10）、util/（9）、v1/（9，兼容层）、persistent-pty/+pty/（12，三平台）

### 第三梯队（一目录一 Service）

account/credential/integration/oauth/kv/modal/reference/ripgrep/vcs/formatter/instance/event 各 1 文件；project/workspace/skill/permission/instructions/image/worktree 各 2-4 文件。

### session/ 头部文件（按 LOC）

| 文件 | LOC | 职责 |
|---|---|---|
| compaction.ts | **988** | 上下文压缩——全仓单文件复杂度之王 |
| projector.ts | 810 | 事件 → 读模型投影 |
| runner/publish-llm-event.ts | 680 | LLM 流式事件发布 |
| model-transport.ts | 571 | 模型连接管理 |
| inbox.ts | 550 | 待办队列（准入/幂等/投递模式） |
| stats.ts | 436 | 用量统计 |
| session.ts | 426 | Session.make 工厂 |
| runner/llm.ts | 408 | drain 循环 |
| transfer.ts | 339 | 导出/迁移 |
| runner/step.ts | 321 | 单步执行 |

---

## 3. TUI / app / gui-extensions 内部分布

### tui/src（62.5k）

| 目录 | 文件 | LOC | 备注 |
|---|---|---|---|
| mini/ | 40 | **18,213** | 第二套完整 TUI（LOC 最大子目录！） |
| component/ | 56 | 14,198 | 51 个 dialog/功能组件 |
| routes/ | 28 | 8,930 | 三路由 + session 渲染链 |
| feature-plugins/ | 25 | 5,204 | merman/latex/storybook |
| context/ | 29 | 3,782 | 25+ provider |
| ui/ | 21 | 3,298 | 基础件 |
| 其余 | 85 | ~9,000 | util/plugin/config/theme/prompt |

### app/src（131k）

runtime/ **161 文件**（数据/事件/连接基础设施）> session/ 82 > settings/ 60 > shell/ 59 > composer/ 41 > providers/ 39 > home+new-session+workspaces+servers 84

### gui-extensions/src（845 文件）

browser/ 99 > file/ 88 > ssh/ 81 > terminal/ 79 > wsl/ 78 > updater/ 75 > review/ 73 + details/ 73 > context/ 70 > debug/ 68 > microsoft-office/ 31 > 其余 67——14 个垂直模块，各带 Electron main/renderer 双面。

---

## 4. 测试分布（共 1174 个测试文件）

| 包 | 测试文件 | 备注 |
|---|---|---|
| **core** | 262 | 每 1.6 个源文件 1 个测试（事件溯源引擎的重测试负担） |
| **app** | 252 | Playwright e2e + 性能基准 + 回归 |
| **tui** | 172 | 渲染器测试 + cli surface 测试 |
| session-ui | 41 | Playwright 组件测试 |
| server | 29 | handler 集成 |
| client | 21 | 测包装层（生成物不测） |
| 其余 | ~319 | ai（provider 磁带）、codemode、merman 等 |

测试位置约定（`.opencode/skills` 有专档）：单元测试与源码同包内跑 `bun test`；根目录禁止跑测试（guard）。

---

## 5. 三个复杂度热点（结构视角的结论）

1. **`core/session/compaction.ts`（988 行）**：摘要 + 保留窗口 + 指令 epoch + 溢出恢复交织——改动前必读 `specs/v2/session.md` 的 Compaction 节。
2. **`tui/mini/`（18k LOC）**：与主 UI 完全平行、不共享代码的第二套 TUI——改 UI 时容易只想着主 UI 而漏掉 mini。
3. **`gui-extensions`（845 文件）**：桌面功能全量载体，Electron IPC 双面结构使文件数天然翻倍。

## 相关篇

- [package-map.md](2_package-map.md) — 这些数字对应的包职责与依赖
- [overview.md](1_overview.md) — 各子系统内部结构
- TODO: session-engine.md — compaction 热点的源码级走读
