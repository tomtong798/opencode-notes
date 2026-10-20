# 数据层详解（事件是真相源，表是投影：Drizzle+SQLite 双平台存储 + 事件总线 + KV）

> 一句话：**Core 的持久化由一条中心化的 `Bus.publish` 管线承担——durable 事件在同一 SQLite 事务里追加 `event` 表、跑投影写读模型表，提交后再分发内存订阅；表只是投影，`event`/`event_sequence` 才是真相源。**
> 代码基线：v2.0.30，dev 分支（`e25b7abf89`）；行号实测（2026-10-19）。
> 权威出处：`specs/v2/event-stream-architecture.md`
> [返回 README](README.md) | 前置 [0_effect-infra.md](0_effect-infra.md) | 关联 [../5_timeline/2_server-init.md](../5_timeline/2_server-init.md)（数据库初始化时序）

## 0. 模块地图

```
packages/core/src/
├─ database/                        ★ 存储底座：Drizzle + SQLite + 迁移
│  ├─ database.ts          122      ★ Database Service：组装 pragma + 迁移 + 锁
│  ├─ drizzle.ts             1      转口 → drizzle/index.js
│  ├─ drizzle/                     自研 Effect×Drizzle 桥接（原独立包内迁）
│  │  ├─ index.ts            5      出口：EffectLogger + effect-sqlite/*
│  │  ├─ effect-sqlite/
│  │  │  ├─ driver.ts       78      makeWithDefaults：SqlClient→Drizzle db
│  │  │  └─ session.ts     241      事务 session 实现
│  │  └─ sqlite-core/effect/        select/insert/update/delete/query/db/
│  │     （7 个文件 198~421 行）    全部返回 Effect 的查询构造器
│  ├─ sqlite.ts             97      平台无关的 Connection/Client 工厂
│  ├─ sqlite.bun.ts        103      ★ bun:sqlite 适配（#sqlite bun）
│  ├─ sqlite.node.ts       100      ★ node:sqlite 适配（#sqlite node/default）
│  ├─ sqlite.workerd.ts    224      Durable Object SQLite 适配（#sqlite workerd）
│  ├─ path.ts              95      absoluteColumn/directoryColumn/pathColumn 路径列
│  ├─ schema.sql.ts        10      Timestamps 公共列（time_created/time_updated）
│  ├─ schema.gen.ts       283      生成物：全新库的一次性建表 SQL
│  ├─ migration.ts        147      ★ 迁移引擎：bootstrap / applyOnly
│  ├─ migration.gen.ts    104      生成物：50 个迁移的有序清单
│  ├─ migration/           50 个    手写 TypeScript 迁移（13~173 行）
│  ├─ v1-migration.ts       2      #v1-migration 转口
│  ├─ v1-migration.bun.ts 1135      V1→V2 数据迁移（读旧库搬 session/message）
│  └─ v1-migration.noop.ts 11      非 bun 平台的空实现
├─ bus.ts                 908      ★ 事件总线：publish/subscribe/log/replay
├─ event/
│  └─ sql.ts              26      ★ event / event_sequence 两张表的定义
└─ kv.ts + kv/
   ├─ kv.ts               92      ★ KV Service：get/set/remove/scan
   └─ sql.ts               9      kv 表定义（key 主键 + JSON value）
```

表定义分散在领域模块的 `*.sql.ts`（如 `session/sql.ts` 170+ 行），`database/` 只持有基础设施。`packages/core/package.json:33-37` 用 subpath imports `#sqlite` 做 `workerd`/`bun`/`node` 三平台条件解析。

## 1. 心智模型

**事件是真相源，表是投影。** 读模型表（`session_v2`、`session_message`…）全部由投影器（`bus.project` 注册）在 durable 事件提交的同一事务里写入；崩溃后理论上可以清空投影、重放 `event` 表重建（`Bus.replay` 就是这条路径的逐条版）。

```
Session.prompt(...)
   │
   ▼
Bus.publish(def, data, { commit? })
   │
   ├─ ① KeyedMutex 按 aggregateID 串行化（进程内）
   │
   ├─ ② db.transaction（behavior: "immediate"）────────────┐
   │     读 event_sequence.seq = latest                      │
   │     seq = latest + 1，构造 durable envelope             │
   │     prepareRoutes（Location 路由快照，暂存不生效）        │
   │     ③ 逐个跑 projectors（写 session_v2 等读模型表）      │  同一
   │     ④ options.commit(seq)（调用方的本地投影）            │  SQLite
   │     ⑤ UPSERT event_sequence（seq 推进）                 │  事务
   │     ⑥ persist 时 INSERT event 表（durable 追加）         │
   │     提交或整体回滚                                      ┘
   │
   ├─ ⑦ 提交后：committed.route()（路由快照才生效）
   │     唤醒该 aggregate 的 durable 订阅（log follow 尾随）
   │     notify()：listeners → typed PubSub → live PubSub
   │
   └─ ⑧ Server event-feed（一个全局 listen）过滤 + 编码 + SSE 帧
          （SSE 帧格式与队列语义归 ../5_timeline/4-5，此处不展开）
```

关键顺序：**投影（③④）先于事件行写入（⑥），全部同事务**——事件行是"提交凭证"，投影和凭证要么都落盘要么都没有。订阅分发（⑦）严格在事务提交之后，因此慢订阅者或失败的 listener 永远不会撕开数据库状态。ephemeral 事件（无 `durable` 定义）跳过 ①-⑥，只做路由 + notify。

## 2. Database Service 与双平台适配

### 2.1 组装（database.ts）

`databaseLayer`（`database.ts:33-53`）四步：`makeDatabase`（Drizzle 桥接）→ pragma → 按库加锁跑 `DatabaseMigration.apply` → 暴露 `{ db }`。启动时序（谁 provide 它、Global data 目录解析）归 [../5_timeline/2_server-init.md](../5_timeline/2_server-init.md)，本篇只讲 pragma。

### 2.2 pragma 清单及依据（database.ts:39-47）

| pragma | 行 | 含义 |
|---|---|---|
| `journal_mode = WAL` | :40 | 写前日志：读写不互斥，崩溃恢复粒度为一事务 |
| `synchronous = NORMAL` | :41 | WAL 下每个事务只 fsync checkpoint，不做每提交 fsync——本地单机库的吞吐/耐久折中 |
| `busy_timeout = 5000` | :42 | 多连接（server+嵌入实例）锁竞争时等 5s 再报 SQLITE_BUSY |
| `cache_size = -64000` | :43 | 64 MiB 页缓存（负数=KiB） |
| `wal_checkpoint(PASSIVE)` | :44 | 启动时被动 checkpoint，控制 WAL 残留 |
| `foreign_keys = ON` | :47 | SQLite 默认关外键，必须显式开（event→event_sequence 的 CASCADE 依赖它） |

全部包在 `supportsTuningPragmas` 守卫里（:39）：workerd 的 Durable Object SQLite 只允许白名单 pragma，`journal_mode` 等直接抛错（`sqlite.workerd.ts:25-32` 注释），且外键由 `SQLITE_DEFAULT_FOREIGN_KEYS=1` 恒开、`PRAGMA foreign_keys` 被拒（`sqlite.workerd.ts:34-38`），故还有 `supportsForeignKeyToggle = false` 守卫。两个能力位都从 `#sqlite` 平台入口导出（bun/node 均为 `true`，见 `sqlite.bun.ts:10-14`、`sqlite.node.ts:10-14`）。

### 2.3 双平台（实为三平台）适配

旧笔记说"sqlite.bun.ts 双平台"，现码是**三个适配文件**，由 `package.json` 的 `#sqlite` 条件导出选择（bun→`sqlite.bun.ts`，node/default→`sqlite.node.ts`，workerd→`sqlite.workerd.ts`）：

- `sqlite.ts`（97 行）是平台无关骨架：`makeConnection`（:27）把 `run/runValues` 包成 effect/sql 的 `Connection`；`makeClient`（:50）用信号量实现单连接上的事务互斥（`transactionAcquirer` :64-71），编译器固定为 SQLite 方言。
- `sqlite.bun.ts`：`native.query()` 每次编译语句，`statement.all/values`，支持 `export`/backup 扩展。
- `sqlite.node.ts`：`native.prepare()` + `setReadBigInts`（safeIntegers 对应物），结构同 bun 版。
- `sqlite.workerd.ts`（224 行）：Durable Object `storage.sql.exec`，自带 `transactionSync` 适配。

### 2.4 杂项：文件权限与内存库

`restrictToOwner`（`database.ts:85-98`）在 POSIX 上把 db/wal/shm 全部 chmod 0600（防 token 类数据被同机用户读）；`lockFor`（:59-65）让同一路径的两个实例共享一个 bootstrap 信号量；`:memory:` 每次独立锁（:71-75）。注释 :28-32 解释了锁为何按"被构建的数据库"而非模块作用域——workerd 同 isolate 多 Durable Object 共享模块状态的坑。

## 3. 迁移

### 3.1 引擎（migration.ts）

`apply`（:22-52）分两条路：

- 库里已有 `session`/`session_v2` 表 → 走 `applyOnly` 增量跑 `migration.gen.ts` 清单。
- 库非空但没有 session 表 → die（:31，拒绝在陌生库上建表）。
- **空库** → 单事务里 `schema.up`（一次性建全部表）+ 建 `migration` 表并把 50 个迁移 id 全部标记完成（:34-46）。即新装用户不逐个重放历史迁移。

`applyOnly`（:54-147）是常路径：查 `migration` 表已完成的 id，跳过、逐个跑剩下的；每个迁移在**独立事务**里执行 `up` + 写入完成标记（:111-118），失败即停（:132-140 有 tapError 日志）。`foreignKeys: false` 的迁移在可切平台用 `PRAGMA foreign_keys = OFF` 包裹，在 workerd 退化为 `defer_foreign_keys`（:119-131）。

兼容桥梁：旧安装用 Drizzle 官方 journal（`__drizzle_migrations`），`applyOnly` 会把旧条目换算成新 `migration` 表的种子行（:62-105，两条分支对应有/无 name 列的旧 schema）。

### 3.2 迁移文件长什么样

`ls migration/ | wc -l` = **50 个**，全部手写 TypeScript（`Migration` 类型：`{ id, foreignKeys?, up(tx) }`，`migration.ts:14-18`），排序由 `migration.gen.ts` 的 import 顺序静态决定（m00…m49，时间戳前缀升序）。两个样本：

```ts
// migration/20260323234822_events.ts —— 建事件表（本篇主角的诞生）
const migration = { id: "20260323234822_events", up(tx) { /* CREATE TABLE event_sequence / event */ } }
// migration/20260428004200_add_session_path.ts —— 最小迁移
yield* tx.run(`ALTER TABLE \`session\` ADD \`path\` text;`)
```

大的如 `20260804233008_loose_psylocke.ts`（173 行）做表重构。`schema.gen.ts` 与 `migration.gen.ts` 是生成物（文件名 `.gen`），与 `drizzle/` 下的 `README.md`（35 行）一起说明这套由代码生成工具维护。

### 3.3 v1-migration 的角色

V1→V2 一次性数据迁移：`v1-migration.bun.ts`（1135 行）用 `bun:sqlite` 直连旧库，把旧 `session`/`session_message`/part 逐条 transform 后写入 V2 的 `SessionTable`/`SessionMessageTable`/`KVTable`（import 见文件头 1-25 行），并带 Warning 收集。非 bun 平台用 `v1-migration.noop.ts` 直接返回 `{ status: "completed" }`。`migration.ts:100-104` 注释提到"migrations may read it (the v1 import)"——个别迁移（如 `20260805200742_import_legacy_credentials`，109 行）会调它搬历史凭据。

## 4. 事件总线（bus.ts）

908 行，`@opencode/Bus` Service（:172）。接口面（:139-170）：`publish` / `publishAll` / `subscribe` / `log` / `listen`(deprecated) / `project` / `replay` / `remove` / `claim`。

### 4.1 publish 的精确语义（commitDurableEvent，:284-448）

单条 durable publish 的完整顺序（`publish`:507 → `publishEvent`:450 → `commitDurableEvent`）：

1. **锁**：`durableLocks.withLock(aggregateID)`（:469）进程内按聚合串行；跨进程靠 SQLite 事务。
2. **事务开启**：`db.transaction(..., { behavior: "immediate" })`（:433）——立即拿写锁，避免升级死锁。
3. 事务内依次：读 `event_sequence` 得 `latest`（:321-327）→ 校验 aggregate 字段/owner（:298-313）→ `seq = latest + 1`（:371）→ `persist` 时查重 event id（:380-394）→ `prepareRoutes`（:399，Location 路由先算后用）→ **跑该 versionedType 的全部 projectors**（:400-402）→ `options.commit(seq)`（:403）→ UPSERT `event_sequence.seq = max(old, seq)`（:404-415）→ `persist` 时 INSERT `event` 行（:416-430）。
4. 事务提交后（Effect.uninterruptible 包裹整段，:315）：`committed.route()` 让路由生效（:436-437，注释 :252-253：失败的 move 不能把事件引去错误 Location）→ 唤醒该聚合的 durable follow 订阅（:438-442）→ 单条路径里 `notify(event, true)`（:474）。

**原子性**：投影、commit 钩子、seq 推进、事件追加在一个 SQLite 事务里，任何一步 die 整体回滚——不会出现"投影写了但事件没记"或反之。**分发不原子**：notify 在事务外，listener 失败只被 `observe`（:486-492）捕获记日志，不影响已提交状态。

`publishAll`（:531-669）是批量版：强制同聚合（:566-574），一次事务连续分配 `firstSeq..finalSeq`（:588-592），投影逐个跑完后一次性 UPSERT 尾 seq + 批量 INSERT 事件行（:643-649），提交后逐个 notify（:663）。

`options.commit` 的文档注释（:96）值得抄：*"Local operational projection committed atomically with a new durable event. Not replayed or serialized."*——它是调用方搭事务顺风车写本地状态的口子。

### 4.2 listen / subscribe / log

- `listen`（:873-880）：往 `listeners` 数组 push 一个回调，返回退订 Effect。已标 deprecated（:161），Server event-feed 仍用它挂全局观察者（`server/src/event-feed.ts:86-88`：`make(bus.listen)`），接缝仅此一处——SSE 帧编码、4096 容量 dropping 队列归 [../5_timeline/4_renderer-boot.md](../5_timeline/4_renderer-boot.md)。
- `subscribe`（:748-760）：volatile 直播流。无参=live PubSub 全量；单定义=typed PubSub；多定义=live 流上按 type 过滤。有 Location 上下文时经 `local()`（:726-746）按路由快照过滤。注释（:122-128）明确：只保证"从现在起"，可靠性要配合 `log`。
- `log`（:828-871）：durable 有序读。分页读 `event` 表（`readAfter`:764-806，页大小默认 512 :202），follow 模式先订阅 wake 再读历史（注释 :853-854：重放期间提交的事件要么进历史读、要么走 marker 后的 wake），在重放水位发一个 `log.synced` marker（:857-862），之后尾随新 seq。
- `replay`（:671-701）：带 `seq` 重放一条序列化事件——seq 已被覆盖且内容逐字段相同（`isDeepStrictEqual`，:345-350）则幂等跳过；diverged 则 die（:361-366）。owner 不匹配则静默丢弃（:368-370，多写者场景的 fencing 雏形）。
- `remove`（:703-715）删聚合的全部事件，`claim`（:717-724）改 `event_sequence.owner_id`。

### 4.3 persist 开关与 Location 路由

`persist`（:179, :203）默认 **false**：`event` 表不落行，只有 `event_sequence` 推进 + 投影。生产 server 打开它（`server/src/routes.ts:119`：`Bus.configured({ persist: options.events?.persist })`，来自 `server/src/options.ts:21-23` 的 `events.persist` 布尔配置）；测试普遍 `persist: true`（如 `test/session-projector.test.ts:39`）。`prepareRoutes`（:212-261）维护 sessionID→Location 的内存路由（session.created/moved/forked/deleted 特判），供 `local()` 过滤和事件投递走向。

## 5. 事件溯源（event/）

`event/sql.ts` 全部 26 行，两张表：

| 表 | 列 | 说明 |
|---|---|---|
| `event_sequence` | `aggregate_id` TEXT PK, `seq` INTEGER, `owner_id` TEXT? | 每聚合一行的"单调水位 + 归属者" |
| `event` | `id` TEXT PK, `aggregate_id` TEXT→event_sequence(CASCADE), `seq` INT, `created` INT default 0, `type` TEXT, `data` TEXT(json) | durable 事件行 |

注意现码**没有 `version` 列**——版本编进 `type` 字符串（`versionedType`:87，如 `session.created@2`），事件信封的 version 在内存 envelope（:67-71）里由 durable manifest 定义给出。索引：`(aggregate_id, seq)` 唯一索引 + `(aggregate_id, type, seq)` 二级索引（sql.ts:22-25）。`event` 初见于迁移 `20260323234822_events`（当时无 `created`/`owner_id`/二级索引，后续迁移补齐）。

**重放语义**：`log` 是只读重放（分页 + follow），`Bus.replay` 是写入侧重放（幂等 / divergence 检测 / owner 过滤，见 §4.2）。重放所有权与集群执行所有权分离是 AGENTS.md 的明确约束（见 §8）。

## 6. KV 存储

`kv.ts`（92 行）+ `kv/sql.ts`（9 行）。`kv` 表：`key` TEXT PK、`value` TEXT(json)、`...Timestamps`。Service 四个方法直白：`get`（:41-48）、`set` UPSERT（:49-56，顺带刷 `time_updated`）、`remove`（:57-59）、`scan` 前缀分页（:60-80，`prefixEnd`:85-90 算前缀的字典序上界，limit 夹在 1..1000，多取 1 行判断是否有 next）。

**与 database 的关系**：KV 不是独立存储——layer 直接取 `(yield* Database.Service).db`（kv.ts:39），建表来自 `schema.gen.ts` 的 `kv`（:71-78）。定位是"事件溯源装不下的小键值"：配置缓存、event-logger 之类的小状态（v1-migration 也往它搬数据）。没有事件、没有投影，就是一张普通表。

## 7. 读模型表全景

一句话职责表（逐列细节归 [4_session 篇]；定义集中在 `session/sql.ts`）：

| 表 | 定义处 | 职责 |
|---|---|---|
| `session_v2` | session/sql.ts:22 | Session 聚合快照：目录/标题/agent/model/token 账目，由 `session.created` 等投影维护 |
| `session_message` | session/sql.ts:79 | 可见消息（(session_id, seq) 唯一），`session.inbox.delivered` 投影消费 inbox 行后插入 |
| `session_pending` | session/sql.ts:100 | 待定/执行中辅助状态 |
| `session_inbox` | session/sql.ts:125 | **只存未消费的准入工作项**（enqueued_seq 唯一），delivery 投影同事务删除 |
| `instruction_entry/blob/state` | session/sql.ts:147-167 | 系统指令条目/去重 blob/组装状态 |
| `event` / `event_sequence` | event/sql.ts:4,10 | 真相源（§5） |
| `kv` | kv/sql.ts:5 | 小键值（§6） |
| `account`/`credential`/`permission`/`project`/`workspace`… | 各自 `*.sql.ts` | 凭据、授权、项目与工作区注册表（schema.gen.ts 全量 283 行可查） |

投影注册集中在 `session/projector.ts:447-601+`（`bus.project(SessionEvent.Created, ...)` 等），`SessionEvent.Created` 投影用 `.onConflictDoNothing()` + 无返回行即 `SessionAlreadyProjected` die（projector.ts:183 附近，:447-468）——幂等与"重复即异常"并存。

## 8. 不变量与设计取舍

来自 `packages/core/AGENTS.md` "Session Core" 第一条与代码注释：

1. **"Keep durable events minimal"**：durable 事件只记不可归约的新事实，不重复可由有序聚合历史折叠出的状态；消费者要自包含视图时去富化投影/读模型，而不是往事件里塞。这是 `event.data` 只有编码后 payload、派生账目（cost/tokens）走投影 `applyUsage`（projector.ts:58-68）的原因。
2. **投影事务性**：事件 + 投影 + seq 同事务（§4.1），spec `event-stream-architecture.md:25` 把 "transactional projection" 列为 Core 职责；:197 明确 "Transactional projector registration is unrelated and remains unchanged"。
3. **分发顺序**：Core 逐个顺序调用观察者、不 fork（spec :132），保证所有健康订阅者看到同一顺序；Server 侧独立溢出队列（spec :47-64 的 Delivery Law）。
4. **路由晚绑定**：prepareRoutes 算好快照但提交后才应用（bus.ts:252-253 注释）。
5. **重放所有权 ≠ 执行所有权**（AGENTS.md 末条）：`event_sequence.owner_id`/`claim` 服务于重放 fencing，execution claim 是另一个机制。
6. **命名空间约定**：OpenCode 拥有无前缀表；嵌入方可用 `_` 前缀表共存，bootstrap 忽略之（migration.ts:24-27 注释）。
7. **取舍**：WAL+NORMAL 换吞吐；persist 可关（测试/内存库不付事件行存储）；`log` 读到未知 durable type 时跳过而非失败（bus.ts:787-789 注释，游标仍前进）。

## 9. 失败与恢复

- **崩溃恢复**：WAL 模式下未完成事务整体消失——事件行、seq 推进、投影三者同事务，不存在半提交状态。重启后 `DatabaseMigration.apply` 补迁移，`event_sequence` 水位即重放起点。
- **投影幂等**：三层防线——(a) 投影器普遍 `onConflictDoNothing`（session/projector.ts:183,468；session/inbox.ts:312；instruction-state.ts:134,230；workspace.ts、project.ts、permission/saved.ts 同）；(b) `Bus.replay` 对已覆盖 seq 做 `isDeepStrictEqual` 全等校验，相等跳过、相异 die "Replay diverged"（bus.ts:337-366）；(c) persist 下 event id 全库查重（:380-394, :603-624）。
- **事务边界**：每条 publish/publishAll 一个 `{ behavior: "immediate" }` 事务；`Effect.uninterruptible` 包裹（:315, :576）防 fiber 取消撕开半事务；迁移每条独立事务 + 完成标记（migration.ts:111-118）。
- **并发**：进程内 KeyedMutex 按聚合串行（bus.ts:200, :469）；跨实例靠 immediate 写锁 + `busy_timeout`；同文件多实例共享 bootstrap 信号量（database.ts:55-65）。
- **执行层恢复**（顺带指路）：write-ahead execution claim + 启动恢复在 session/execution，非本篇范围。

## 演进注记

对照旧笔记 4.19（2026-08-03，gitee `5_modules/4.19_database/database.md`）：

1. **双平台→三平台**：旧文只写 bun/node；现码 `#sqlite` 多了 `workerd`（Durable Object），且以 `supportsTuningPragmas`/`supportsForeignKeyToggle` 两个能力位替代散落的平台判断。文件仍叫 `sqlite.bun.ts`（旧笔记的文件名结论仍对）。
2. **effect-drizzle-sqlite 独立包→内置目录**：旧文的 `packages/effect-drizzle-sqlite` 现为 `packages/core/src/database/drizzle/`（driver 78 行/session 241 行 + sqlite-core/effect 全家），`drizzle.ts` 只剩 1 行转口。
3. **路径列仍在但扩充**：`path.ts` 95 行，`absoluteColumn/directoryColumn/pathColumn`，Windows 反斜杠规范化（path.ts:5-22）；`session_v2.directory` 即用 `directoryColumn()`（session/sql.ts:35）——旧文"路径列类型"主题延续。
4. **旧文的 schema.sql.ts 是表聚合，现码只剩 10 行 Timestamps**：表定义彻底下沉到领域 `*.sql.ts`，全量建表改由生成的 `schema.gen.ts` 承担。
5. **迁移体系换血**：Drizzle 官方 journal → 自研 `migration` 表 + 手写 TS 迁移（50 个），带旧 journal 换算桥（migration.ts:62-105）。
6. **本篇新增旧文没有的主线**：bus.ts 事件总线 / event 表 / persist 开关 / 幂等重放——旧文写库时事件层还没长成现在的形状（event 表 2026-03-23 才诞生）。

## 相关篇

- [0_effect-infra.md](0_effect-infra.md)：Layer/Service/Global 等 Effect 基建（本篇 `makeGlobalNode`、Semaphore 的底层）
- [../5_timeline/2_server-init.md](../5_timeline/2_server-init.md)：Database/Bus 在 server 启动时的初始化时序
- [../5_timeline/4_renderer-boot.md](../5_timeline/4_renderer-boot.md)：event-feed → SSE 帧 → TUI 消费链
- specs/v2/event-stream-architecture.md：Server 事件流编码/队列的权威决策记录
