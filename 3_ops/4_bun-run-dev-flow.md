# `bun run dev` 执行流程分析

> 工作目录: `/home/donghe/pet/opencode/packages/desktop-electron`
>
> Bun 版本: `1.3.14` (Linux arm64)
>
> TMPDIR: `/home/donghe/tmp`

---

## 一、整体执行链路

```
bun run dev
  │
  ├─ [predev 钩子]  bun ./scripts/predev.ts
  │   ├─ Step A:  bun ./scripts/copy-icons.ts dev
  │   │   └─ 复制图标到 resources/icons
  │   │
  │   └─ Step B:  cd ../opencode && bun script/build-node.ts
  │       ├─ script/generate.ts        ← 拉取 models 数据
  │       ├─ 加载 SQL migration 文件
  │       └─ Bun.build()               ← 构建 Node.js bundle
  │
  └─ [dev 主任务]  electron-vite dev
      └─ 启动 Electron + Vite 开发服务器
```

---

## 二、各步骤详细说明

### 2.1 `package.json` 脚本定义

```jsonc
// /packages/desktop-electron/package.json
{
  "scripts": {
    "predev":  "bun ./scripts/predev.ts",   // "pre" 前缀 → npm 钩子，自动在 dev 前执行
    "dev":     "electron-vite dev",          // 主任务
    "prebuild": "bun ./scripts/prebuild.ts", // prebuild 内容与 predev 相同
    "build":   "electron-vite build"
  }
}
```

`predev` 是 npm 的 **pre-script 钩子**，`bun run dev` 会自动先执行 `predev`，成功后才执行 `dev`。

---

### 2.2 Step A — 图标复制 (`copy-icons.ts`)

```ts
// scripts/copy-icons.ts
import { $ } from "bun"
import { resolveChannel } from "./utils"

const channel = resolveChannel()             // 读取 OPENCODE_CHANNEL，默认 "dev"
const src  = `./icons/${channel}`            // 如 ./icons/dev
const dest = "resources/icons"

await $`rm -rf ${dest}`
await $`cp -R ${src} ${dest}`                // 复制图标资源
```

**输入**: `./icons/dev/` 目录
**输出**: `resources/icons/` 目录
**作用**: 根据构建渠道（dev/beta/prod）复制不同的图标资源（应用图标、托盘图标等）

> 注：`copy-icons.ts` 又 `import { resolveChannel } from "./utils"`，触发 `utils.ts` 的加载。`utils.ts` 也 `import { $ } from "bun"`，这意味着 Bun Shell 在 Step A 就开始初始化了。

---

### 2.3 Step B — 构建 opencode Node.js bundle (`build-node.ts`)

```ts
// /packages/opencode/script/build-node.ts

// 1. 生成 models-snapshot.js
await import("./generate.ts")
//    └─ 从 OPENCODE_MODELS_URL (默认 https://models.dev/api.json) 拉取模型数据
//    └─ 写入 src/provider/models-snapshot.js + .d.ts

// 2. 加载 SQL migration 文件
const migrations = await Promise.all(
  migrationDirs.map(async (name) => {
    const sql = await Bun.file(".../migration.sql").text()
    return { sql, timestamp, name }
  })
)

// 3. Bun.build() — 编译 TypeScript → Node.js ESM
await Bun.build({
  target: "node",
  entrypoints: ["./src/node.ts"],
  outdir: "./dist/node",
  format: "esm",
  sourcemap: "linked",
  external: ["jsonc-parser", "@lydell/node-pty"],
  define: {
    OPENCODE_MIGRATIONS: JSON.stringify(migrations),
    OPENCODE_CHANNEL: `'${Script.channel}'`,
  },
})
```

**作用**: 把 opencode 的 TypeScript 源码编译为可在 Node.js（Electron 主进程）中运行的 ESM bundle。

---

### 2.4 主任务 — `electron-vite dev`

`electron-vite` 是一个集成了 Electron 和 Vite 的构建工具。`dev` 模式会：

1. 启动 Vite 开发服务器（热更新 renderer 进程）
2. 编译 main 进程 (`src/main/`) 和 preload 脚本 (`src/preload/`)
3. 启动 Electron 窗口

---

## 三、Bun tempdir 机制与问题分析

### 3.1 Bun 如何确定 tempdir

Bun 在启动时按以下优先级确定 tempdir：

```
1. $BUN_TMPDIR
2. $TMPDIR          ← 当前环境设为 /home/donghe/tmp
3. $TMP
4. $TEMP
5. /tmp             ← 兜底
```

Bun 会在 tempdir 中创建带哈希前缀的临时文件/目录：

```
.<16-hex-hash>-<pid>.<type>

.hm    — 堆内存文件 (heap memory)
.start — 进程临时目录
.plex  — Plexus 后台任务临时目录
```

### 3.2 发现的问题

`/home/donghe/tmp` 中存在来自之前 Bun 会话的 **stale 残留文件**：

```
.38bfbffeffefef77-00000002.start   ← Bun 进程临时目录（含残留数据）
.3ebfb7fbfaf34ebe-00000001.plex   ← Plexus 目录（含大量 scss 文件）
.5ef3bf76fe7bbf9d-00000000.hm     ← 堆内存文件（空文件，可能持有文件锁）
```

### 3.3 两种异常表现

| 错误 | 根因 |
|------|------|
| `error: bun is unable to write files to tempdir: AccessDenied` | 新 Bun 子进程尝试 `mkdir` 创建 temp 目录时，hash 与已有 stale 目录碰撞，`openat(O_EXCL)` 失败 |
| **卡住无响应**（`$ bun ./scripts/predev.ts` 后无输出） | `.hm` 文件被已崩溃但未清理的旧进程持有 `flock`/`fcntl` 文件锁，新进程在启动时尝试获取同一锁 → **阻塞等待** |

### 3.4 为什么 Step A 可能成功、Step B 失败

`predev.ts` 通过 Bun Shell (`$``) 启动了两个 **独立的子 bun 进程**：`copy-icons.ts` 和 `build-node.ts`。每个子进程独立初始化 tempdir，生成不同的哈希值。哈希碰撞是概率性的——可能一个撞上、一个没撞上。

---

## 四、排查步骤

### Step 1 — 清理僵尸进程

```bash
ps aux | grep -i bun | grep -v grep

# 如果有残留的 bun 进程
pkill -f "bun.*opencode"
pkill -f "bun.*desktop"
```

### Step 2 — 清理 stale temp 文件

```bash
rm -rf /home/donghe/tmp/.*.start /home/donghe/tmp/.*.plex /home/donghe/tmp/.*.hm
ls -la /home/donghe/tmp/   # 验证清理干净
```

### Step 3 — 分步验证

```bash
cd /home/donghe/pet/opencode/packages/desktop-electron

# 3a: 单独跑 copy-icons
bun ./scripts/copy-icons.ts dev

# 3b: 单独跑 build-node
cd ../opencode
bun script/build-node.ts

# 3c: 最小 Bun Shell 测试
bun -e "import { \$ } from 'bun'; console.log('ok'); await \$`echo hello`; console.log('done')"
```

### Step 4 — 完整验证

```bash
cd /home/donghe/pet/opencode/packages/desktop-electron
bun run dev
```

### 决策树

```
Step 2 清理后直接 Step 4
  ├─ 成功 → ✓ 根因确认：stale 文件冲突
  │
  ├─ Step 3c 卡住 → Bun Shell 初始化失败，升级 Bun
  │
  ├─ Step 3a 卡住/失败、Step 3c 正常 → utils.ts 或 copy-icons 依赖问题
  │
  └─ Step 3b 卡住/失败、Step 3a 正常 → build-node.ts 依赖问题
      └─ 检查 generate.ts 网络请求是否超时（models.dev）
```

---

## 五、快速修复命令

```bash
# 一键修复（杀进程 + 清 temp + 重试）
pkill -f "bun.*opencode" 2>/dev/null
pkill -f "bun.*desktop" 2>/dev/null
rm -rf /home/donghe/tmp/.*.start /home/donghe/tmp/.*.plex /home/donghe/tmp/.*.hm
cd /home/donghe/pet/opencode/packages/desktop-electron && bun run dev
```
