# 资源治理（Resource Limits）

> 一句话：**opencode v2 没有一个集中的"限流器"，而是把限额当作局部不变量散植在每个模块的边界上——工具输出有界化（2000 行/50KiB）、事件订阅者 4096 容量断连、附件 20MB、数据库按文件名信号量串行——每条限额都贴着"谁生产、谁负责截断"的原则长在代码里。**
> 代码基线：v2.0.30，dev 分支（commit `e25b7abf89`）；行号实测（2026-10-22）。
> 权威出处：`packages/core/src/tool-output.ts`、`packages/server/src/event-feed.ts`、`packages/core/src/session/runner/retry.ts`、`packages/core/src/database/database.ts` 等散点（全部见 §1 表）。
> [返回 README](README.md) | 前置 [../6_modules/9_tool.md](../6_modules/9_tool.md) | 关联见文末"相关篇"。

本篇是"非功能机制详解"的收口篇：各散点篇只讲了自己域内的限额，这里把它们拼成一张全景表，再深挖三个代表机制（输出有界化、事件背压、并发预算），并从时间与磁盘两个维度补齐治理面。

---

## 0. 恐惧清单

限额不是拍脑袋定的，每条背后都是一个具体恐惧。走读代码时反推出来的失控场景：

1. **工具输出撑爆上下文**：`grep -r` 在 monorepo 里吐 50 万行，全文进 LLM 上下文 → 单条消息超过整个窗口。→ 2000 行/50KiB 截断 + 全量落盘（§2a）。
2. **工具输出撑爆磁盘**：截断落盘本身是"每工具调用写一个文件"，无人清理则 `~/.local/share/opencode/tool-output/` 无限增长。→ 7 天保留 + 每小时清扫（`tool-output.ts:15,129-137`）。
3. **并发把 provider 打限流**：8 个附件同时物化、多 session 并发 drain，请求风暴触发 429。→ 物化并发 8（`prompt.ts:54`）+ 退避预算封顶 retry-after 15min（`retry.ts:32`）。
4. **单个慢客户端拖死事件流**：SSE 订阅者卡住，发布端若阻塞则整个 server 冻结。→ dropping 队列 4096，满则只断开该订阅者（`event-feed.ts:8,64-68`）。
5. **大附件 OOM**：20MB base64 附件解码后 ×1.33，多个并发直接顶爆堆。→ server 端 20MB 硬拒（`prompt.ts:203`）、CLI 路径 10MB（`run.ts:60`）、websearch 响应 256KiB（`websearch/mcp.ts:7`）。
6. **递归 subagent 指数爆炸**：subagent 再开 subagent，深度无界则一次失控 fork 出 2ⁿ 个 session。→ 深度默认 1（`tool/plugin/subagent.ts:125`，`experimental.subagent_depth` 可调）。
7. **恶意/故障 provider 用 retry-after 拖死 session**：返回 `retry-after: 6h`。→ `RETRY_AFTER_MAX = 15 minutes` 封顶（`retry.ts:31-32`，注释原话 "a hostile or buggy retry-after cannot stall a session for hours"）。
8. **MCP 服务器单帧炸弹**：stdio 帧无界读 → 一条 JSON 消息吃 1GB。→ `MAX_FRAME_BYTES = 16 MiB`（`mcp/stdio.ts:15,121`）。
9. **日志无限增长**：调试级日志常年追加。→ 50MiB 上限就地压缩到 25MiB、每小时检查（`observability/logging.ts:9-11`）。
10. **僵尸恢复循环**：进程反复崩溃重启，恢复尝试永不停止。→ `DEFAULT_MAX_ATTEMPTS = 10`（`restart.ts:33`），耗尽后置 `aborted` 不再自动恢复（`restart.ts:18-21`）。

---

## 1. 全景限额表

核心交付物。grep 口径：`MAX|LIMIT|capacity|timeout|budget|concurrency|RETENTION`。全部行号 2026-10-22 实测于 v2.0.30 dev（`e25b7abf89`）。

### 1.1 核心域（工具 / 输出 / 上下文）

| 域 | 机制 | 限额值 | 定义位置 | 超限行为 |
|---|---|---|---|---|
| 工具输出 | 行数/字节双截断 | 2000 行 / 50 KiB | `core/src/tool-output.ts:13-14` | 截断 + 全量落盘 + marker 指路（§2a） |
| 工具输出 | 落盘保留期 | 7 天 | `core/src/tool-output.ts:15` | 每小时按 mtime 清扫删除 |
| read 工具 | 每次读取上限 | 2000 行 / 单行 2000 字符 | `core/src/tool/read-filesystem.ts:14,18` | 分页提示，模型自己翻页 |
| shell 输出 | 复用工具输出限额 | 默认同上（可 config 覆盖） | `core/src/shell.ts:236-237` | 同工具输出；落盘文件 `sh_*.out` 同样 7 天（`shell.ts:33,106`） |
| webfetch | HTML→Markdown 上限 | 5 MiB（内容区再减 64KiB） | `core/src/tool/html-markdown.ts:50-51` | 截断转换结果 |
| websearch | MCP 响应上限 | 256 KiB | `core/src/plugin/websearch/mcp.ts:7` | 报错 "response exceeded" |
| HTTP 工具 | 响应体有界收集 | `collectBoundedResponseBody` | `core/src/tool/http-body.ts:4` | 按 content-length/增量封顶 |
| 压缩摘要 | 工具输出重放上限 | 1250 字符/条 | `core/src/session/compaction.ts:100` | 截断进摘要请求 |
| 压缩摘要 | 摘要输出 token | 32,000（SUMMARY_OUTPUT_MAX） | `core/src/session/model-request.ts:53` | 请求级 min 封顶（见 [7_compaction](../6_modules/7_compaction.md)） |
| LLM 输出 | 输出 token 封顶 | 256,000（fallback 32,000） | `core/src/session/model-request.ts:49-51` | `min(catalog, MAX)` |
| 错误体 | provider 错误正文 | 2000 字符 | `ai/src/route/executor.ts:87` | 截断加省略号 |
| LLM 事件 | 输入摘录 | 2048 字符 | `core/src/session/runner/publish-llm-event.ts:32` | 事件里只放 excerpt |

### 1.2 附件与输入边界

| 域 | 机制 | 限额值 | 定义位置 | 超限行为 |
|---|---|---|---|---|
| server 附件 | 尺寸硬拒 | 20 MB | `core/src/session/prompt.ts:203` | AttachmentError，不回显字节 |
| server 附件 | 物化并发 | 8 | `core/src/session/prompt.ts:54` | 排队等信号量 |
| CLI --attach | 文件上限 | 10 MB | `cli/src/run/run.ts:60` | 命令行直接失败 |
| TUI 本地附件 | 读取上限 | 20 MB | `tui/src/component/prompt/local-attachment.ts:5` | 读取失败提示 |
| Web 端 | 内联上限 | 20 MB（走 path 路由绕过） | `app/src/composer/attachments/attachments.ts:268` | 大媒体改传文件路径 |
| 图片内联 | data URL 上限 | 256 KiB | `ai/src/protocols/replicate-images.ts:14` | 改走非内联 source |
| 指令条目 | 单条 value | 256 KiB | `schema/src/instruction-entry.ts:30`（core 侧检查 `session/instruction-entry.ts:136`） | 报错指明实际/上限字节数（见 [6_context](../6_modules/6_context.md)） |
| 会话标题 | 标题/上下文/首消息 | 100 / 8,000 / 2,000 字符 | `core/src/session/title.ts:20-22` | 截断 |

### 1.3 事件流与客户端

| 域 | 机制 | 限额值 | 定义位置 | 超限行为 |
|---|---|---|---|---|
| server 事件 | 订阅者队列容量 | 4,096 | `server/src/event-feed.ts:8` | 只断开溢出者（§2b） |
| 客户端事件 | 本地订阅者缓冲 | 4,096 | `client/src/shared-events.ts:25` | 该订阅者报错终止，连接可复用 |
| SSE 心跳 | 保活注释 | 15 秒 | `server/src/handlers/event.ts:21` | — （防中间盒断连） |
| 客户端 | 空闲看门狗 | 45 秒 | `client/src/solid/connection.ts:35` | abort "Event stream stalled" → 重连 |
| 客户端 | 连接超时/重连延迟/前台阈值 | 2s / 1s / 20s | `client/src/solid/connection.ts:33-38` | 重连循环 |
| TUI | 输出宽限 | SHELL_OUTPUT_GRACE_MS = 1500 | `tui/src/mini/stream-v2.transport.ts:397` | 放弃等待 live 事件（见 [6_tui-cli]） |
| TUI | 结构化回退长度 | 4096 | `tui/src/mini/tool.ts:1247` | 截断展示 |

### 1.4 时间与重试预算

| 域 | 机制 | 限额值 | 定义位置 | 超限行为 |
|---|---|---|---|---|
| provider 重试 | 次数预算 | 10 次（2s 指数起，10s 封顶） | `core/src/session/runner/retry.ts:45-48` | 放弃步骤，报错 |
| provider 重试 | retry-after 封顶 | 15 分钟 | `retry.ts:31-32` | 取 `max(delay, min(retryAfter, 15min))` |
| provider 重试 | 超时类专项预算 | 3 次 | `retry.ts:60` | 提前放弃（§3） |
| 恢复 | 重启恢复尝试 | 10 次 | `core/src/session/execution/restart.ts:33` | 置 `aborted` 永不自动恢复 |
| WebSocket | 连接超时 | 15 秒 | `core/src/session/model-transport.ts:23` | 回退 HTTP |
| WebSocket | 流空闲超时 | 30 分钟（推理模型可静默） | `model-transport.ts:25` | idle-timeout 断流 |
| WebSocket | 连续失败降级 | 5 次 exchange | `model-transport.ts:27` | 该 session 留在 HTTP |
| WebSocket | 通道轮换 | 55 分钟 | `model-transport.ts:22` | 重建 channel |
| MCP | 启动/目录/执行超时 | 30s / 30s / 12h | `core/src/mcp/client.ts:34-36` | 失败/取消工具调用（见 [14_mcp](../6_modules/14_mcp.md)） |
| MCP | 进程终止宽限 | 1 秒 | `mcp/client.ts:37` | 强杀 |
| 表单 | 待答保留 | 10 分钟 | `core/src/form.ts:8` | 表单过期 |

### 1.5 并发与存储

| 域 | 机制 | 限额值 | 定义位置 | 超限行为 |
|---|---|---|---|---|
| 数据库 | busy_timeout | 5000 ms | `core/src/database/database.ts:42` | SQLite 等锁后再报错 |
| 数据库 | 按文件名串行 | Semaphore(1)/文件 | `database.ts:57-71` | 跨层共享同一把锁（§2c） |
| 文件事务 | KeyedMutex | 按 path 键 | `core/src/file-mutation.ts:60,70` | 排队 |
| inbox | KeyedMutex | 按 sessionID | `core/src/session/inbox.ts:59` | 排队 |
| workspace | KeyedMutex | 按 workspace ID | `core/src/workspace.ts:91` | 排队 |
| session | 执行串行化 | run-coordinator Map | `core/src/session/run-coordinator.ts:6,71` | 同 key 合并/排队 |
| 文件保留清理 | 删除并发 | 8 | `core/src/file-retention.ts:21` | 排队 |
| skill 发现 | 目录/文件并发 | 4 / 8 | `core/src/skill/discovery.ts:13-14` | 排队 |
| subagent | 嵌套深度 | 默认 1 | `core/src/tool/plugin/subagent.ts:125` | ToolFailure 提示调 `experimental.subagent_depth`（见 [8_agent](../6_modules/8_agent.md)） |
| PTY | 输出缓冲 | 2 MiB | `core/src/pty.ts:12` | 丢头部保尾部 |
| PTY/shell | 已退出进程保留 | 25 个 | `pty.ts:15`、`shell.ts:32` | 淘汰最旧 |
| 日志 | 单文件上限 | 50 MiB → 压到 25 MiB | `util/src/observability/logging.ts:9-10` | 每小时就地裁头（按行对齐） |
| 桌面扩展包 | 包体/条目数 | 100 MB / 4,096 条 | `desktop/src/main/extension/archive.ts:8-10` | 拒绝安装 |
| 桌面 i18n | 原生载荷 | 64 KiB | `app/src/runtime/i18n/desktop-native.ts:319` | 跳过（返回 undefined） |
| Web 持久化 | LRU 缓存 | 500 条 / 8 MiB | `app/src/runtime/persistence/storage.ts:51-53` | 淘汰 |
| Web 会话列表 | 最近会话/目录存储 | 50 / 30 | `app/src/runtime/server/global-sync/types.ts:72,78` | 截断列表 |
| shell 扫描 | 输入长度/替换深度 | 64 KiB / 32 | `core/src/shell/scan.ts:89-90` | 拒绝展开 |
| 搜索 | ripgrep 子匹配 | 100 | `core/src/ripgrep.ts:20` | 截断 submatches |
| glob/search | 默认结果上限 | 100 | `core/src/filesystem.ts:69` | 截断 |

### 1.6 codemode 沙箱（解释器内）

| 域 | 机制 | 限额值 | 定义位置 | 超限行为 |
|---|---|---|---|---|
| 字符串/数组长度 | 16 MiB / 10,000,000 | `codemode/src/interpreter/limits.ts:7,9` | 抛错 |
| pending promise / 值深度 | 10,000 / 32 | `limits.ts:11,13` | 抛错 |
| 调用栈深度 | 10,000 | `codemode/src/interpreter/interpreter.ts:376` | 抛错 |
| console 序列化深度 / schema 渲染深度 | 32 / 8 | `stdlib/console.ts:30`、`tool-schema.ts:41` | 以 "…" 截断 |

（codemode 细节属独立域，本表只收口，不展开。）

---

## 2. 代表机制深挖

### 2a. 工具输出有界化全链

把 §1.1 前三行串成一条链：**生成 → 截断 → 落盘 → 保留期清理**。

```
工具 execute() 返回 Tool.NormalizedResult
 │
 ├─ 未超限（lines ≤ 2000 且 bytes ≤ 50KiB）
 │   └─ tool-output.ts:75-76  仅标记 metadata.truncated=false，原样返回
 │
 └─ 超限
     │
     ├─ ① 预算切分（tool-output.ts:78-85）
     │     逐行累加字节，先到 maxLines 或 maxBytes 者止
     │     → kept[] = 保留的行
     │
     ├─ ② 全量落盘（tool-output.ts:86-88）
     │     ~/.local/share/opencode/tool-output/tool_<id12>.out
     │     写完整 text（未截断原文）
     │
     ├─ ③ marker 替换（tool-output.ts:89-112）
     │     content 数组按字符预算切片，
     │     尾部插入 marker：
     │     "[showing lines 1-N of M; full output saved to <file>]"
     │     metadata.truncated=true, metadata.outputPath=<file>
     │     （模型可再用 read 工具分页读该文件 → 有界但可回溯）
     │
     └─ ④ 保留期清理（tool-output.ts:129-144，cleanupNode）
           每小时（Schedule.spaced 1h）扫 tool-output/ 目录
           只匹配 /^tool_[0-9a-f]{12}/
           → FileRetention.cleanup（file-retention.ts:8-26）
             mtime < now - 7天 → remove，8 并发删除
```

要点：

- **双维度取交集**：行数与字节同时满足才算未超限（`tool-output.ts:75`），先到的预算生效。
- **幂等保护**：已带 `truncated` 元数据的结果直接跳过（`:68`），不会二次截断。
- **限额可配置**：`config/plugin/tool-output.ts:20` 把 config 的 `max_bytes`/`max_lines` 注入 State editor，但常量默认就是 2000/50KiB。
- **shell 同构**：shell 输出走同一套默认（`shell.ts:236-237`），落盘文件名 `sh_<id>.out`，同样 7 天（`shell.ts:33`）。也就是说 read 工具的 2000 行/50KiB（`read-filesystem.ts:14,18`）与输出截断是两个独立关卡：前者限制单次读多少，后者限制工具结果进上下文多少。

### 2b. 事件流背压闭环

慢消费者问题的完整解法：**发布端永不阻塞，慢的那一个被牺牲，然后它自己重连重取**。

```
                    ┌─ 发布侧（server 进程内）────────────────┐
Bus.publish(event) ─┤ EventFeed.make.publish (event-feed.ts:48-69)
                    │  ① isOpenCodeEvent 过滤（内部事件不占容量）
                    │  ② encode 一次（frame = "data: {json}\n\n"）
                    │  ③ 遍历 subscribers: Set<Queue>
                    │     Queue.offerUnsafe(q, frame)
                    │       ├─ 有空位 → continue（发布端零等待）
                    │       └─ 满 → subscribers.delete(q)
                    │            Queue.failCauseUnsafe(SubscriberOverflowError{capacity:4096})
                    └──────────────────────────────────────────┘
                          │ Queue.dropping(4096)   ← event-feed.ts:76
                          ▼
                    ┌─ 订阅侧（每个 SSE 连接一个）──────────────┐
                    │ Stream.fromQueue(q) → SSE response        │
                    │ handlers/event.ts:11-32                   │
                    │   首帧 server.connected                    │
                    │   merge 15s 心跳（haltStrategy:"left"）     │
                    └──────────────────────────────────────────┘
                          │ 溢出 → 队列 fail
                          ▼
                    ┌─ 客户端恢复 ──────────────────────────────┐
                    │ client/solid/connection.ts                │
                    │  · 空闲看门狗 45s：任何字节（含心跳注释）    │
                    │    都 touch 续期（connection.ts:87-92）     │
                    │  · 流断 → 重连循环（reconnectDelay 1s，     │
                    │    连接超时 2s）                            │
                    │  · 前台恢复阈值 20s：休眠唤醒后静默超过它    │
                    │    判定半开连接直接重建（:36-38）            │
                    │  · 重连后依赖 server.connected + 增量状态   │
                    │    接口重取，SSE 只做通知不做事实来源        │
                    └───────────────────────────────────────────┘
```

要点：

- **dropping 而非 sliding/bounded**：`Queue.dropping`（`:76`）意味着满员后新事件被丢弃而不是挤掉旧事件——对事件流来说"旧事件已渲染、新事件补得上"由客户端重连重取兜底。
- **只杀慢的**：`publish` 对每个队列独立 offer，一个溢出不影响其它订阅者（测试名就叫 "fails only the subscriber that exceeds its lag capacity"）。
- **编码失败更激进**：encode 抛错会 `fail` 清空**全部**当前订阅者（`:41-46`）——因为无法确定哪个事件损坏，宁可全员重连（也有对应测试）。
- **客户端二次防御**：`client/src/shared-events.ts:25,88-91` 在迭代器一层还有本地 4096 缓冲，消费者（如 Solid store）卡住时同样自断，且连接在无订阅者时整体 stop。

### 2c. 并发预算体系

三层结构：**数值并发 → 会话级串行 → 键级互斥**。

```
┌─ 层1：数值并发（Effect.forEach { concurrency: N }）────────────┐
│  附件物化 8        session/prompt.ts:54                        │
│  目录扫描 8        shell.ts:104 / git.ts:484,505 / project.ts:441 │
│  保留清理 8        file-retention.ts:21                        │
│  skill 发现 4/8    skill/discovery.ts:13-14（目录4 文件8）        │
│  （instruction 插件读文件 unbounded —— 文件量由 discovery 界定，  │
│    config/plugin/instruction.ts:83）                            │
└────────────────────────────────────────────────────────────────┘
                          │
┌─ 层2：会话执行串行（run-coordinator）──────────────────────────┐
│  session/run-coordinator.ts:6 注释：                            │
│  "Serializes execution for each key while allowing different    │
│   keys to run concurrently"                                    │
│  实现：executions: Map<Key, Execution>（:71）                    │
│  同一 session 的 resume/wakeup 合并（join），trampoline 防栈涨（:85）│
│  → 这是"single in-flight LLM call per session"的结构保证        │
└────────────────────────────────────────────────────────────────┘
                          │
┌─ 层3：键级互斥（KeyedMutex / Semaphore）───────────────────────┐
│  KeyedMutex.makeUnsafe<K>()（effect/keyed-mutex.ts:21-33）       │
│    每个 key 一个 Semaphore(1)，引用计数 users，归零即释放 Map 项   │
│  · file-mutation.ts:60,70   按 path —— 文件事务串行              │
│  · session/inbox.ts:59      按 sessionID —— 入队/交付互斥        │
│  · workspace.ts:91          按 workspace ID                     │
│  · config/plugin/{skill,instruction}.ts:34-35 各自 Semaphore(1) │
│  · codemode/tool.ts:85      解释器输出锁                        │
│                                                                │
│  database 信号量（database.ts:57-71）：                          │
│    locks: Map<filename, Semaphore> —— 同一 SQLite 文件跨 Layer   │
│    共享一把串行锁；":memory:" 每实例一把（Semaphore.make(1)）。    │
│    配套 PRAGMA busy_timeout = 5000（database.ts:42）跨进程兜底。  │
│  workerd 变体同构：sqlite.workerd.ts:171 Semaphore.make(1)。     │
└────────────────────────────────────────────────────────────────┘
```

要点：

- **为什么数据库要信号量**：better-sqlite 同连接不是线程安全复用的，且迁移/写入需串行；按文件名共享锁让多个 Layer 装配（不同服务各自拿 Database）不会并行写同一文件。busy_timeout 只管**跨进程**锁竞争，进程内靠 Semaphore。
- **KeyedMutex 不是全局锁**：不同 key 完全并行，只有同 key 排队；`workspace.ts:361-366` 的 TODO 还点名 filesystem/search 与 run-coordinator 在手搓同一 join/cancel 形状，未来可能抽公共件。
- **层1 的 8 不是配额系统**：就是 `Effect.forEach` 的并发参数，没有全局预算池——opencode 的哲学是"每处局部限速足够小，加起来就不会打爆"。

---

## 3. 时间维度治理

把 §1.4 的 timeout 按"等谁"分类，并解释它们与重试预算的关系：

| 类别 | 等谁 | 值 | 位置 |
|---|---|---|---|
| 建连 | provider WebSocket | 15s | `model-transport.ts:23` |
| 建连 | 客户端→server SSE | 2s | `connection.ts:33` |
| 空闲 | LLM 流静默 | 30min（注释：推理模型可静默数分钟） | `model-transport.ts:25` |
| 空闲 | 客户端事件流 | 45s 看门狗（心跳 15s < 45s，留 3 倍余量） | `connection.ts:35` + `event.ts:21` |
| 保活 | SSE 心跳注释 | 15s | `handlers/event.ts:21` |
| 轮换 | WebSocket 通道 | 55min | `model-transport.ts:22` |
| MCP | 启动 30s / 目录 30s / 执行 12h / 终止 1s | `mcp/client.ts:34-37` |
| 数据库锁 | busy_timeout | 5s | `database.ts:42` |
| 宽限 | TUI shell 输出 | 1500ms | `stream-v2.transport.ts:397` |
| 表单 | 等用户作答 | 10min（pending 时 infinity） | `form.ts:8,114` |

**与重试预算的关系**（`retry.ts` 全文走读）：

- 调度是 `min(指数2s, 间隔10s)` 与 `recurs(10)` 的复合（`:45-48`）：间隔 2→4→8→10…，最多 10 次重试（初始 1 次 + 10 次 = 11 次尝试），全程等待约 84s（加 jitter 67-101s）。
- `retry-after` 只在 RateLimit/ProviderInternal 两种错误上生效，且被 15min 封顶（`:34-38`）；与退避取 `max`——尊重 provider 但不受制于它。
- **超时专项预算**（`:58-60` 注释直说动机）：一次 Timeout 尝试本身已经等了若干分钟，若也允许 10 次，死 provider 能占住 step 接近一小时；所以 `timeouts <= 3` 一到就放弃（决策在 `:84`）。
- **重启恢复是另一本账**（`restart.ts:33`）：进程崩溃后的自动 resume 最多 10 次尝试，且计数**持久化**（"counts every resume durably"，对应测试 `session-execution.test.ts:318`）；耗尽后发 `RESUME_EXHAUSTED`（type: aborted，`:18-21`），并在恢复 prompt 里注入 CONTINUE_AFTER_SERVER_RESTART 让模型接着干而不重做。
- 前端 sending 侧的失败预算独立存在：WebSocket 连续 5 次 exchange 失败（`model-transport.ts:27,190`）后该 session 降级 HTTP，不再反复撞坏通道。

---

## 4. 磁盘治理

谁负责打扫，打扫什么，多久一次：

| 打扫者 | 对象 | 策略 | 位置 |
|---|---|---|---|
| ToolOutput cleanupNode | `tool-output/tool_*.out` | 每小时，mtime > 7 天删 | `tool-output.ts:129-144` + `file-retention.ts` |
| Shell cleanup layer | `storage/<project>/sh_*.out` | 每小时，7 天 | `shell.ts:104-110` |
| Logging trim | 日志文件 | 每小时检查，>50MiB 就地裁头到 25MiB（按行对齐，flock 互斥，锁 5min 过期） | `util/src/observability/logging.ts:9-14,86-120` |
| DesktopCli.cleanStages | CLI 各版本 stage 目录 | 每次启动，删掉非当前版本目录（Windows 删不掉则下次再试） | `desktop/src/main/service/cli-stages.ts:5-29` |
| PTY 已退出表 | 内存结构 | 上限 25 条 LRU 淘汰（非磁盘，但同属资源回收） | `pty.ts:15,234` / `shell.ts:32,378` |
| 扩展安装 | 旧版本 | 安装新包前校验 100MB/4096 条，替换式安装不留旧版 | `extension/archive.ts:8-10` |

compile cache：桌面 CLI 的 staging 属 launch 时清理；core 侧 `LayerNode.compile`（`instance.ts:159`）是进程内图编译，不落盘，无清理义务。日志压缩的设计选择值得一提：**不 rotate（不产生历史文件）而是就地截断**——`logging.ts:7-8` 注释说明动机是避免"两个 trim 各自按对方已缩的 size 计算切点"，用 `<file>.trim` 锁串行化。

---

## 5. 失效场景矩阵

| 限额 | 调低会怎样（误伤） | 调高会怎样（失守） | 边界绕过 |
|---|---|---|---|
| 工具输出 2000 行/50KiB | 大文件 diff 被切掉关键 hunk，模型多轮 read 翻页浪费 token | 单条工具结果逼近窗口，压缩触发提前 | 多 content 块拼接总量按字符预算切（`tool-output.ts:92-111`），但 file 类型 content 原样通过（`:96-99`）——file 引用本身就是绕过设计 |
| 订阅者 4096 | 高频事件（流式 part）下慢 UI 频繁断连抖动 | 慢客户端长期积压 4096×帧大小内存 | 客户端 solid connection 的 pending/flush（10ms 批）在 upstream，绕不过 dropping 队列；但 encode 失败全员断连（`:41-46`）是故意的"宁断不腐" |
| 附件 20MB | 正当视频/数据集传不上去（TUI 本地读同 20MB） | base64 ×1.33 放大，8 并发物化瞬时堆峰值 ~213MB | CLI 10MB 与 server 20MB 不一致：CLI 拒得更早；data URL 解码有 canonical base64 校验（`prompt.ts` decodeDataURL） |
| subagent 深度 1 | 合法的"研究→写作"两级嵌套要改 experimental 配置 | 每层 ×N 个并行 subagent，深度 d 即 N^d 个 session | 深度靠遍历 parentID 链计算（`subagent.ts:112-124`），session 树持久化所以跨进程准确；但 subagent 内再 spawn 的工具并发不受深度限制 |
| retry 10 次 | 抖动型 429 提前放弃，用户看到失败 | 指数上限 10s/次，总等待有界（~84s+），调高次数线性拉长 | `RETRY_AFTER_MAX` 15min×10 次 ≈ 2.5h 理论最坏等待——但仅限 provider 明示 retry-after 的错误 |
| 恢复 10 次 | 长崩溃循环后用户需手动 resume | 无限僵尸恢复循环 | 计数持久化，删 session 才清零（`session-execution.test.ts:588` 验证删 session 确认 marker） |
| MCP 执行 12h | 无（几乎不会误伤） | 真·挂死工具占住 step 12h | 启动超时 30s 与执行超时独立；catalog 30s 意味着新 server 工具最多晚 30s 出现 |
| busy_timeout 5s | 高并发写锁等待 5s 后报 SQLITE_BUSY | 锁等待霸占 fiber | 进程内 Semaphore 先串行化了，5s 只兜跨进程（如 CLI 与 server 同时开） |
| 指令条目 256KiB | 超大生成物指令（如完整 schema 文档）被拒 | instruction state 无限膨胀进每条请求 | 报错信息带实际字节数（`instruction-entry.ts:136-140`），可诊断；条目**数量**无上限——只限单条 |

---

## 6. 测试覆盖

限额相关测试实测（`packages/*/test`，bun test）：

- `core/test/tool-output.test.ts`（155 行，8 个 `it.live`）：超限落盘+有界预览、字节限下行数报告、混合 content 顺序、跳过已截断结果、未超限不改动、尾换行不算一行、字节限下的尾换行省略、mtime 兜底——覆盖 §2a 全分支。
- `server/test/event-feed.test.ts`（5 个 `it.effect`）：单订阅者溢出只断它一个、内部事件不占容量、编码失败断开全员但后续订阅者可用——§2b 的两个关键行为都有断言。
- `core/test/session-runner-retry.test.ts`：transport 超时 3 次后停止重试、其它瞬态错误保留完整预算——§3 专项预算。
- `core/test/session-execution.test.ts:318,379,588`：恢复尝试持久计数且不消耗正在恢复的 claim、shell owner 恰好一次投递恢复通知、删除 session 时确认 recovery marker——§3 恢复预算。
- `core/test/session-prompt.test.ts:511`：拒绝超限内联附件且**不回显字节**（错误信息只说 20971520 上限）。
- `core/test/tool-subagent.test.ts:351,360`：深度 1 到限报错、配置 `subagent_depth: 2` 放行嵌套。
- `core/test/session-compact.test.ts`、`session-compaction-transport.test.ts`：摘要输出预算（SUMMARY_OUTPUT_MAX 侧）。
- 缺口：并发常量（8/4）无直接测试（靠 Effect 语义保证）；日志 trim、cli-stages 在各自包有测试（`cli-stages.test.ts`）；PTY 2MiB 缓冲依赖运行时行为，未见专门断言。

---

## 演进注记

- **新增域（2026-10 系列）**：workspace 体系刚落地，`workspace.ts:359-366` 留有三个 TODO——boot janitor、~23h 安全快照轮换、把 run-coordinator/search 手搓的 keyed 共享尝试抽成公共件。资源治理面预计随之扩张（快照保留期会是新限额）。
- **`experimental.subagent_depth`** 是少数暴露给用户的限额旋钮；工具输出 max_lines/max_bytes 也走 config（`config/plugin/tool-output.ts:20`），图片内联 max_base64_bytes 同理（`config/plugin/image.ts:24`）——"常量默认 + config 覆盖"是限额可配置化的通行模式。
- codemode 沙箱限额（limits.ts）独立成文件，预示解释器域会继续加护栏。
- 客户端 45s 看门狗 + 20s 前台阈值 + 15s 心跳的组合在 `connection.ts:35-38` 注释里有明确推导（心跳间隔的整数倍、休眠唤醒半开检测）——时间类限额之间是协同设计而非孤立数字。

## 相关篇

- [9_tool.md](../6_modules/9_tool.md) §7：工具输出截断在工具管线中的位置
- [5_prompt-chain.md](../5_timeline/5_prompt-chain.md)：附件 8 并发物化时序
- [14_mcp.md](../6_modules/14_mcp.md)：MCP timeout 三层
- [6_context.md](../6_modules/6_context.md)：instruction entry 256KiB
- [7_compaction.md](../6_modules/7_compaction.md)：SUMMARY_OUTPUT_MAX 与窗口预算
- [8_agent.md](../6_modules/8_agent.md)：subagent 深度
- [1_data-layer.md](../6_modules/1_data-layer.md)：数据库层与 busy_timeout
