# 日志体系与位置（Log Locations）

> 一句话：**两套独立日志体系——后端（CLI/TUI/server）单文件追加 + 原地压缩，桌面（Electron）按启动时间戳分目录 + 7 天清理；核心代码在 `util/src/observability/logging.ts`。**
> 基于代码实测：v2.0.30，`dev` 分支（2026-10-16）。
> [返回索引](README.md) | 姊妹篇：[deployment-layout.md](2_deployment-layout.md)

---

## 1. 两套体系总览

| | 后端日志（core fileLogger） | 桌面日志（Electron） |
|---|---|---|
| 代码 | `packages/util/src/observability/logging.ts` | `packages/desktop/src/main/native/logging.ts` |
| 位置 | `{XDG_DATA_HOME}/opencode/log/` | `{userData}/logs/{启动时间戳}/` |
| 文件 | `opencode-{channel}.log`（**全机单文件，所有 opencode 进程共写**） | `main.log` + `renderer.log` + `network.netlog`（每次启动新目录） |
| 策略 | 原地压缩（超 50MB 砍头保 25MB） | 7 天后整目录删除 |
| 排障入口 | `OPENCODE_LOG_LEVEL` / `OPENCODE_PRINT_LOGS=1` | electron-log + netlog |

---

## 2. 后端日志（fileLogger）

### 路径构造（logging.ts:59-62）

```ts
export function file(local = true, channel = "local") {
  if (!local) return path.join(Global.Path.log, "opencode.log")
  return path.join(Global.Path.log, `opencode-${channel.replace(/[^a-zA-Z0-9._-]/g, "-")}.log`)
}
```

- Linux/macOS：`~/.local/share/opencode/log/opencode-{channel}.log`（channel 来自 `OPENCODE_CHANNEL`，如 dev/beta/prod）
- **设计要点**：全机一个文件按 channel 分文件、多进程 `O_APPEND` 共写——TUI、后台服务、CLI 命令的日志交织在同一文件，靠行内结构化字段区分进程

### 原地压缩（不是轮转）

- 触发：每 1 小时检查，超 `LOG_MAX_BYTES`（50MB）则砍头保约 `LOG_KEEP_BYTES`（25MB），按行边界取整
- **为什么不是 rename 轮转**：其他进程以 O_APPEND 持有该文件，rename 会让它们写进孤儿文件——所以用跨进程 `mkdir` 锁（5 分钟 stale 超时）做原地截断

### 环境变量

| 变量 | 作用 |
|---|---|
| `OPENCODE_LOG_LEVEL` | DEBUG/INFO/WARN/ERROR，默认 INFO |
| `OPENCODE_PRINT_LOGS=1` | 同时把结构化日志写到 stderr |

---

## 3. 桌面日志（Electron）

### 目录结构（logging.ts:200-204）

```
{userData}/logs/{YYYYMMDDTHHMMSS}/   ← 每次启动一个时间戳目录
  main.log          ← 主进程
  renderer.log      ← 渲染进程
  network.netlog    ← 网络日志（上限 20MB）
{userData}/Crashpad/                 ← 崩溃转储
```

`userData` 按平台与通道（`packages/desktop/src/main/constants.ts` 的 APP_ID）：

| 平台 | 路径 |
|---|---|
| macOS | `~/Library/Application Support/ai.opencode.desktop/`（dev 通道 `.dev`、beta `.beta`） |
| Linux | `~/.config/ai.opencode.desktop/` |
| Windows | `%APPDATA%\ai.opencode.desktop\` |

### 清理

启动时后台删除 7 天前的旧 run 目录（`MAX_LOG_AGE_DAYS`）。

### 调试包导出（debug bundle）

桌面导出诊断包时**同时收集**两处服务端日志根：XDG 数据目录（`~/.local/share/opencode/log/`）与 `{userData}/opencode/log/`（`serverLogRoots()`）。

---

## 4. SDK / 嵌入式形态

`packages/sdk/src/logging.ts`：SDK 嵌入时日志走宿主提供的 `LogWriter`（Effect 层）——**不落盘**，由宿主应用决定去留。

---

## 5. 快速排障速查

```bash
# 后端日志在哪（Linux/macOS）
ls ~/.local/share/opencode/log/

# 实时看 + 调级别
OPENCODE_LOG_LEVEL=DEBUG opencode …
tail -f ~/.local/share/opencode/log/opencode-dev.log

# 桌面：找最近一次启动的日志
ls -t ~/Library/Application\ Support/ai.opencode.desktop/logs/ | head -1   # macOS

# 服务进程是否活着（注册文件即发现契约）
cat ~/.local/state/opencode/service.json
```

## 相关源码

- `packages/util/src/observability/logging.ts`（fileLogger、压缩、级别）
- `packages/desktop/src/main/native/logging.ts`（桌面日志、清理、serverLogRoots）
- `packages/util/src/global.ts`（`Global.Path.log` 定义）
- `packages/sdk/src/logging.ts`（宿主 LogWriter）
