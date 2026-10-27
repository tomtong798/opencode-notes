# Subagent 子系统详解（主会话经工具调用派生子会话：派生 → 后台作业 → 完成回流 → 崩溃恢复）

> 一句话：**subagent = 主会话里一次工具调用所派生的"子 Session"——父经 `subagent` 工具创建/续接子会话（parentID 指向父）、注入 prompt、注册为 Job；前台模式 `jobs.block` 阻塞等待子会话 drain 完成并把最后一条 assistant 文本作为 Tool.Success 回流，后台模式立即返回、完成后以 synthetic 消息通知父会话；重启后由 restart.ts 的 recoverSubagent 收养孤儿子会话。**
> 代码基线：v2.0.30，dev 分支；行号实测（2026-10-19）。
> 权威出处：`packages/core/src/tool/plugin/subagent.ts`、`session/subagent-job.ts`、`session/subagent-completion.ts`、`session/execution/restart.ts`；规范 `specs/v2/session.md`（执行声明）。
> [返回 README](README.md) | 前置 [4_session.md](4_session.md)（聚合/inbox/drain）、[9_tool.md](9_tool.md) §4.9（工具输入 schema 与权限点） | 关联 [../5_timeline/6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md)（前端事件归属）、[17_job.md](17_job.md)（Job 注册表本体）

---

## 0.5 通俗完整描述（先读这段再看细节）

主会话里模型调用 subagent 工具时，会**派生一个子会话**（parentID 指回父）：子会话有自己的 inbox、自己的执行循环、完整的 LLM 对话——比如"用 explore 只读探查一下这个库的结构"，探查在子会话里独立进行，**完成后最终结论回流父会话**（作为工具结果），中间过程不占父窗口。这就是上下文工程里"subagent 隔离"杠杆的落地。

它是三个既有机制的薄组合（433 行编排层）：**子会话**（Session 派生，parentID 数据库列）+ **Job 注册表**（后台任务身份，Job ID=子会话 ID）+ **双回流通道**（前台：父的工具调用等它完成拿结果；后台：完成时以"通知消息"形式注入父会话，失败也保留会话 ID 可续问）。

几个关键语义：**深度守卫**默认 1（子不能再派孙——防指数爆炸，可配置）；**权限继承**——子继承父 agent 的 deny 规则（曾出过 Plan Mode 越权事故后补的）；**中断传播**——父被中断会传递给子；**崩溃恢复**——重启后孤儿子会话被恢复子系统认领续跑（有预算上限）；前端归属——TUI/CLI 沿 parentID 链判定"这个子会话的事件归我管"。

一句话：**子会话=独立的上下文容器；派生轻（三个机制组合）、隔离强（过程不进父窗口）、回收全（结果回流+恢复认领）——"分身干活，正身省心"**。
---

## 0. 模块地图

核心三文件仅 433 行——subagent 是"组合 Session + Job 既有机制"的薄编排层，而非独立执行引擎：

```
packages/core/src/
├─ tool/plugin/subagent.ts ★ 派生工具本体（311 行）
│   ├─ Input/Output schema + description（:29-62）
│   ├─ execute：深度守卫→agent 解析→权限→建/续子会话→prompt→Job（:108-256）
│   ├─ execute.before hook：清洗空 model/sessionID 入参（:271-280）
│   └─ context/compaction/generate hook：往工具描述里追加可用 subagent 清单（:282-309）
├─ session/
│   ├─ subagent-job.ts ★ SubagentJob 服务：Job 包装 + 完成通知（76 行）
│   ├─ subagent-completion.ts ★ 文本提取 + 结果投递（46 行）
│   └─ execution/
│       └─ restart.ts ★ 崩溃恢复（recoverSubagent :138-190；总 240 行）
├─ job.ts ★ 依赖：Job 注册表（block/background/wait/pendingBackground；Recovery 联合含
│         kind:"subagent" 分支 :20-25，Background 持久化前缀 "job.background/" :37）
├─ session/sql.ts    session_v2 表 parent_id 列 + parent 索引（:31, :72）
├─ session/store.ts  parentID 过滤查询 + releaseChildClaims（:112, :225-240）
├─ session/runner/llm.ts  settleStaleToolCalls：重启后孤儿工具结算（:362-381）
└─ （消费侧）
packages/tui/src/mini/
├─ stream-v2.subagent.ts ★ TUI subagent tracker（1141 行）
├─ stream-v2.transport.ts  tracker 接线：main/foreign/hydrate 三入口（:538, :990, :1031-1035）
└─ footer.subagent.tsx     底栏 tab 渲染（269 行）
packages/cli/src/run/noninteractive.ts  ownsSession 归属链（:141-156）
```

防重复约定：subagent 工具的输入 schema 字段表、权限 assert 细节归 [9_tool.md](9_tool.md) §4.9；TUI 事件消费全景归 5_timeline/6——本篇只各取一小节作印证。

---

## 1. 心智模型

subagent 不是独立的进程或线程模型，而是**三个既有机制的组合**：

1. **子会话**：一个普通 Session，唯一差别是 `parent_id` 列指向父会话（`sessions.create({ parentID: context.sessionID, ... })`，subagent.ts:187-193）。子会话有自己完整的 inbox 准入、drain 执行、事件流、消息表行——与任何会话同构。
2. **Job 注册表项**：以 `childSessionID` 为 Job ID 注册（subagent-job.ts:45-63），复用 Job.Service 的 block/background/wait/持久化恢复全套语义。
3. **回流通道**：前台走 `jobs.block` 的返回值；后台走 `sessions.synthetic` 往父会话投一条 synthetic 消息（subagent-completion.ts:37-44）。

```
        父会话（主 runner drain 中）
          │ assistant 消息里的 tool call: subagent{agent,description,prompt,model?,sessionID?,background?}
          ▼
   ┌─────────────────────────────┐
   │ subagent.execute            │
   │ 1. 深度守卫（沿 parentID 链上溯计数）      │
   │ 2. agents.resolve + primary 禁用 + permission.assert │
   │ 3. 建/续子会话（parentID=父）             │
   │ 4. sessions.prompt 注入任务文本           │
   │ 5. SubagentJob.start → Job 注册          │
   └───────┬─────────────────────────────┘
      foreground │              background
           ▼                      ▼
   jobs.block(子ID)         subagents.background + 立即返回
           │                      │
           ▼                      ▼
   ┌──────────────────┐    完成时 SubagentCompletion.deliver
   │ 子会话 drain      │    → sessions.synthetic(父ID, <subagent …>文本)
   │ （自有 runner/    │              │
   │  inbox/事件流）   │              ▼
   └────────┬─────────┘    父会话 inbox 收到通知 → 下轮 drain 读到
            │
            ▼
   Job.run: sessions.messages 取最后完成的 assistant
   → SubagentCompletion.text → Tool.Success(output)
```

关键理解：**子会话与父会话仅通过两条窄通道交互**——(a) 派生时的一次 `sessions.prompt`；(b) 完成时的一次结果回流（Tool.Success 或 synthetic）。运行期间二者完全解耦，父的 LLM 看不到子的过程（除非前台工具的 progress 事件）。

共享与私有的边界：子会话**共享** Location 服务（同一 project/workspace 的工具注册表、权限、文件系统视图），但**私有** inbox、消息历史、事件流——即"同舞台、独立剧本"。

---

## 2. 派生流（subagent.ts execute 逐段）

### 2.1 深度守卫（:110-133）

execute 第一步先沿 parentID 链**向上**走到底，数出嵌套深度：

```ts
// subagent.ts:117-133（摘）
let current = parent
let depth = 0
while (current.parentID) {
  depth++
  current = yield* sessions.get(current.parentID) ...
}
const limit = Config.latest(yield* config.entries(), "experimental")?.subagent_depth ?? 1
if (depth >= limit)
  return yield* new ToolFailure({ message: `Subagent depth limit reached (${limit}). ...` })
```

默认 `subagent_depth = 1`：subagent 不能再派生 subagent。链上任何一环查不到（父会话被删）直接 ToolFailure。注意这是**运行时检查**而非 schema 校验——深度是会话拓扑属性，只有执行时才知道。

### 2.2 agent 选型与权限（:134-151）

- `agents.resolve(input.agent)`——工具入参 `agent` 字段是 agent ID（如 `explore`）；解析失败报 `Unknown agent`。
- `agent.mode === "primary"` 拒绝——主 agent 递归当 subagent 会自我嵌套。
- `permission.assert({ action: "subagent", resources: [agent.id], save: [agent.id], source: {type:"tool",...} })`——权限点按目标 agent 收口（细节归 9_tool.md §4.9 / 11_permission.md）。

### 2.3 续接与切换（:153-182）

入参带 `sessionID` 时是**续接**语义：先校验 `existing.parentID === context.sessionID`（别人的孩子不能抢，:164-167）。续接时若换了 agent，则镜像 create 语义做 switch：显式 model 覆盖 > 新 agent 的 model > 保持原样（注释 :169-170 明说了这条优先级链），用 `sessions.switchAgent/switchModel` 落到子会话行。

### 2.4 模型解析与子会话创建（:168, :184-198）

`Model.Ref.parse` 校验 `providerID/modelID#variant` 格式（:75-98，ToolFailure 报错带可用变体列表）。最终模型三元优先：

```ts
// subagent.ts:184
const model = override ?? agent.model ?? parent.model
```

新建子会话由 **execute 自己设 parentID**（`sessions.create({ parentID: context.sessionID, title: input.description, agent, model })`，:187-193）——parentID 不在任何 hook 或 projector 里推导，唯一写点就是这里（外加 fork 场景写 fork_session_id，projector.ts:455 把 `event.data.parentID` 投到 `parent_id` 列）。`title` 直接用 3-5 词的 `description`，所以子会话在会话列表里可读。

### 2.5 prompt 注入与后台标记（:200-232）

`background` 为 true 且是新建时，prompt 带 `resume: false`——**只准入不唤醒**（admit-only），避免立即触发 drain（见 4_session.md 的 inbox 准入/唤醒分离）。文本注入有个微妙的差异：

```ts
// subagent.ts:208-211（摘）
text: existing === undefined
  ? ["You are a subagent spawned by another session.", input.prompt].join("\n")
  : input.prompt,
```

新会话首条 prompt 前置一句身份声明；续接会话**不加**——续接时子会话已有上下文，再注入身份反而污染。随后 `context.progress({ sessionID: child.id, status: "running" })`（:201）让前端工具部件立刻能挂子会话视图。

`background=true` 路径（:229-232）：`subagents.background(recovery)` 注册后台作业后立即返回 `backgroundResult`——一段明确指示模型"**不要 sleep/轮询/查进度，去做不重叠的工作**"的输出文本（:19-27）。这是用 prompt 工程约束父模型行为的例子。

### 2.6 描述动态增强（:282-309）

`ctx.session.hook("context"|"compaction"|"generate")` 在每次组装上下文时，把**当前 agent 可用**（非 primary、非 hidden、Permission.evaluate 非 deny）的 subagent 清单按字母序追加到工具 description 尾部。效果：模型看到的 subagent 工具描述永远与权限视图一致——被 deny 的 agent 对模型不可见，从源头减少无效调用。

### 2.7 入参清洗（:271-280）

`execute.before` hook 把空字符串 `model`/`sessionID` 从 input 里删掉——provider 有时会把 optional string 参数填成 `""`，schema 的 optionalKey 挡不住空串，这里在执行前洗掉。

---

## 3. 后台作业语义（subagent-job.ts）

`SubagentJob` 是 Session/Job 之上的一个**很薄的全局节点**（`Node.makeGlobalNode`，:72-76；deps: Session.node + Job.node）。接口只有三个方法（:12-16）：`start` / `background` / `notify`。

### 3.1 start：Job 包装（:45-63）

```ts
// subagent-job.ts:45-63（摘）
start: (recovery) =>
  jobs.start({
    id: recovery.childSessionID,      // Job ID == 子会话 ID
    type: "subagent",
    title: recovery.description,
    recovery,                          // 持久化恢复凭据
    run: Effect.gen(function* () {
      yield* sessions.resume(recovery.childSessionID)   // 唤醒子会话 drain
      const messages = yield* sessions.messages({ sessionID: ..., order: "desc", limit: 20 })
      const assistant = messages.find((m) =>
        m.type === "assistant" && m.time.completed !== undefined && m.error === undefined)
      return SubagentCompletion.text(assistant)
    }),
  })
```

要点：

- **Job ID 即 childSessionID**。这解释了 subagent.ts:203-204 的注释：`Job.start` join 一个已在跑的 Job 会跳过 run effect；续接运行中的子会话天然幂等，而 `sessions.prompt` 的默认 wake 要么启动 idle 子会话、要么 steer 运行中的——所以 prompt 准入放在 job 外、由 execute 直接做。
- **run 的产出就是最终文本**：resume 等 drain 跑完，取最近 20 条消息里最后一条"完成且无错"的 assistant，用 `SubagentCompletion.text` 抽 text part。注意这里与 restart.ts 的细微差异：job.ts run 用 `find`（desc 序第一条命中=最新），restart.ts:177 用 `findLast`（asc 序）——语义等价。
- **recovery 持久化**：`{ kind:"subagent", parentSessionID, childSessionID, agent, description }`（subagent.ts:220-226 构造）经 Job.Background schema（job.ts:20-25）以 `job.background/<notificationID>` 前缀落 KV（job.ts:37, :200），重启后 `jobs.pendingBackground` 读回。

### 3.2 notify：一次性完成观察者（:30-42）

```ts
const key = `${recovery.childSessionID}:${startedAt}`   // 每代 job 一个 key
if (notifications.has(key)) return
```

`Set<string>` 去重——同一子会话被多次 background/续接，每代（按 started_at）只投递一次完成通知。观察 fiber 用 `Effect.forkIn(scope, ...)` + `Effect.setContext(context)` 挂到服务自己的 scope（注释 :24-26：job 与观察者生命周期长于派生它的插件，不能继承调用方的 location context）。`jobs.wait` 无超时等 Job done，拿到 info 后交给 `SubagentCompletion.deliver`。

### 3.3 并发限制

**核心侧没有 subagent 数量上限**——并发由 Job.SessionCoordinator 的会话调度天然决定（不同 Session 可并发 drain）。真正的上限在前端消费侧：TUI tracker 的 `FAMILY_LIST_LIMIT = 100`（stream-v2.subagent.ts:44, :230——超限直接不 admit 新 child 进视图）。深度方向只有 §2.1 的 `subagent_depth`。

---

## 4. 完成检测与结果回流（subagent-completion.ts）

全文件 46 行，两个导出：

**`text(message)`**（:10-19）：非 assistant 或无非空 text part → 常量 `NO_TEXT = "Subagent completed without a text response."`；否则全部 text part 以 `\n\n` join。**完成检测 = "找 drain 结束时最后一条 completed 且 error===undefined 的 assistant 消息"**——不解析事件、不看状态机，纯读模型查询。

**`deliver`**（:21-46）：把 Job 终态翻译成父会话的一条 synthetic 输入：

```ts
// subagent-completion.ts:31-44（摘）
const text =
  status === "completed" ? (output ?? NO_TEXT)
  : status === "error"   ? (error ?? "Subagent failed")
  : "Subagent cancelled"
yield* sessions.synthetic({
  ...(input.notificationID ? { id: input.notificationID } : {}),   // 幂等键
  sessionID: recovery.parentSessionID,
  ...(input.resume === false ? { resume: false } : {}),            // 只通知不唤醒
  text: `<subagent sessionID="${childID}" state="${status}" description="${desc}">\n${text}\n</subagent>`,
  metadata: { source: "subagent", childID, agent, state },
})
if (input.notificationID) yield* jobs.completeBackground(input.notificationID)
```

三条设计：

1. **统一信封**：回流文本包在 `<subagent sessionID=… state=…>` 标签里——前台 Tool.Success 的 content 同样是这个格式（subagent.ts:260-263）。父模型无论从工具结果还是从后台通知读到，都拿到同一形状、且**永远带 sessionID**（失败分支也保留，subagent.ts:245-251 注释明说这是为了让模型能续接失败的孩子）。
2. **synthetic + 指定 id**：`notificationID` 复用为 synthetic inbox 项 id → 天然幂等（4_session.md：同 id 重投被 reconciled 掉），重启恢复时不会双投。
3. **resume 标志**：父会话正被挂起恢复时（§7），通知 `resume:false` 只准入不唤醒，避免恢复期被完成通知搅动。

**两条回流路径汇总**：前台 = `jobs.block`（subagent.ts:234-240）→ Tool.Success/ToolFailure；后台 = `deliver` → synthetic → 父下轮 drain 读到。`block` 还挂了 `Effect.onInterrupt`：父的 fiber 被中断时**同时** `sessions.interrupt(child)` + `jobs.cancel(child)`——这就是前台模式的中断传播点。

---

## 5. 父子关系数据模型

### 5.1 表结构（session/sql.ts）

```ts
// session/sql.ts:31（摘）
parent_id: text().$type<SessionSchema.ID>(),
// :72
index("session_v2_parent_idx").on(table.parent_id),
```

nullable、**无外键约束**（对比 project_id 的 `references(... onDelete: "cascade"")`，:24-28）——父会话可先于子被删，子成为"孤儿行"而不是级联消失；一致性靠运行时守卫（§2.1 链上溯查不到即报错；§7 恢复期校验后清理）。另有 `fork_session_id` 是 fork 关系，与 parentID 语义无关，勿混。

### 5.2 写入与读取

- **唯一写点**：`SessionEvent.Created` 携带 parentID → projector 插行（projector.ts:447-463，`parent_id: event.data.parentID` :455）。
- **级联查询**：`SessionStore.list` 支持 `parentID` 过滤——`parentID === null` 转 `isNull`（查顶层会话），否则 `eq`（查某父的直接孩子）（store.ts:108-113）。是**单层过滤**而非树查询：要整棵树得像深度守卫那样自己沿链走（subagent.ts:119-128）或像 CLI ownsSession 那样递归（§8.2）。
- **恢复期专用**：`store.listSuspended()` 只看 `isNotNull(parent_id)` 的行（store.ts:198）——子会话执行声明不参与顶层恢复；`releaseChildClaims`（store.ts:225-240）批量清掉**不在可恢复名单里**的子会话的执行声明（time_suspended/resume_attempts 归零），防止孤儿 claim 卡住统计。
- **解码**：`session/info.ts:21` 行→Info 时 `parent_id ? SessionSchema.ID.make(...) : undefined`。

### 5.3 标题小细节

runner 的 drain 循环里 `!selected.session.parentID && SessionTitle.isUntitled(...)`（runner/llm.ts:177）——**只有顶层会话**才做自动标题生成；子会话标题保持工具给的 description 不被覆盖。同理 runner 识别 subagent 工具 metadata 里的 sessionID（llm.ts:371）用于孤儿结算（§7）。

---

## 6. 中断传播

中断**不是**事件级"传播"，而是三个显式动作点：

| 场景 | 机制 | 出处 |
|---|---|---|
| 前台 block 被中断 | `Effect.onInterrupt` → `sessions.interrupt(child)` + `jobs.cancel(child)` | subagent.ts:234-240 |
| 父会话整体 interrupt | 只中断父自己的 drain；子会话是**独立 Session**，进程内协调器按 sessionID 各自持有所有权链（execution.ts:27-30），不级联 | session.ts:219, :462 |
| 中断后的工具结算 | 父 drain 重启时 `settleStaleToolCalls` 把 streaming/running 的 subagent 工具 call 结算成 `Tool.Failed{type:"aborted"}`，错误信息**带上子 sessionID** | runner/llm.ts:362-381 |

方向性总结：前台模式中断向下传（onInterrupt 显式砍孩子）；后台模式**不传**——父被打断时后台子会话照跑，完成后 synthetic 通知照投（这正是后台语义）。反向：子会话失败/取消对父的影响就是 §4 的 error/cancelled 文本回流，父模型自行决定重试（续接 sessionID）或放弃。

---

## 7. 崩溃恢复（restart.ts recoverSubagent）

进程死亡时未完结的后台 subagent Job 留在 KV（`job.background/*`）。重启后 `resumeSuspendedSessions`（restart.ts:193-230）扫 `jobs.pendingBackground`，先处理 shell 再处理 subagent（排序 :208 保证 shell 通知先于子会话首个模型请求被 admit）。

`recoverSubagent`（:138-190）逐作业处理：

```
读回子会话行
  ├─ 子不存在 / parentID 不匹配 / 父不存在
  │    → completeBackground(通知) 仅清账，不投递                    (:143-147)
  ├─ Job 已有终态（进程死前已完成但通知未消费）
  │    → notify(background) 直接投递存量结果                         (:158-161)
  ├─ 子会话在本进程已 active（别处已恢复）
  │    → return（避免双跑）                                          (:162)
  ├─ prepareResume 失败（重试预算耗尽）
  │    → notify({status:"error", error: RESUME_EXHAUSTED})           (:163-166)
  └─ 可恢复 → jobs.start(id, run: execution.resume(子) → 取最后
       assistant → text) → jobs.background → fork jobs.wait → notify  (:168-189)
```

三个配套动作（:193-221）：

1. **suspended 集合**：listSuspended（有 claim 的顶层会话）∪ 恢复中的子会话 ID——投递时若父在集合里，`resume:false` 只准入不唤醒（notify 包装 :149-156）。
2. **releaseChildClaims(children)**（:205）：可恢复名单**之外**的子会话 claim 清零——重启后孤儿 claim 是脏数据。
3. **恢复完再看 ownership**（:219-220 注释）：后台完成会唤醒父，所以只能在恢复之后再查本地所有权，避免误判。

注意恢复的子会话以 `jobs.background` 身份续跑（不是前台 block——父的 drain 已经中断了，没人 block）。若父会话本身可恢复，`prepareResume` 后 `execution.resume` 会重启父 drain，届时子完成通知经 inbox 正常送达。

---

## 8. 前端归属印证（只读消费侧）

### 8.1 TUI subagent tracker（stream-v2.subagent.ts）

tracker 把"主会话的 subagent 工具调用"与"子会话的事件流"在客户端拼成一张家族视图。三个入口（transport.ts 接线）：

- **`main(sdk, event)`**（transport.ts:1035 → subagent.ts:967-1002）：只看主会话的工具事件。从 `tool.progress/success/failed` 的 **metadata 里抠 sessionID**（`childSessionID()` :188-196：须 `ses` 前缀且 status ∈ {running, completed}）→ `admitChild` 进家族表；`tool.input.started/called` 记 pendingCalls 以便把调用入参（label/description）与 metadata join（:206-214）。`tool.success` 且仍 running → 标记 background。
- **`foreign(sdk, sessionID, event)`**（transport.ts:1031 → :1004-1025）：事件 source ≠ 主会话时先查 children 表——认识就直接 `reduce` 进对应 child（hydrate 进行中则先缓冲，防乱序）；不认识则触发 `discover`（session.get 反查其 parentID 是否指向主会话），发现期间事件缓冲在 pendingEvents，防止"快子会话在发现完成前就结束丢事件"（:253-263 注释）。
- **`hydrate(next)`**（transport.ts:990 → :1026-1102）：连接/重连后用主会话消息快照重建状态，对活跃 child 逐个 `hydrateChild`（拉子会话消息补 80 条窗口，:401-446）。

界限控制（:42-50）：每 child 最多 80 消息/80 帧、家族表 100 上限、无关会话记忆 bounded（checked 集合）、发现并发 8。归属逻辑纯靠 **tool metadata.sessionID + session.parentID**，服务端没有专门事件——印证 §5 的"父子关系就是数据列，没有隐式通道"。

### 8.2 CLI ownsSession（noninteractive.ts:141-156）

```ts
// Subagents run in child sessions; their asks and questions belong to this run too. Other
// sessions on a shared server (e.g. the TUI's) must be left alone.
const ownsSession = (sessionID) => ... session.parentID ? ownsSession(session.parentID) : false
```

CLI run 在共享 server 上订阅全量事件流，用**沿 parentID 链递归上溯到根是否等于 input.sessionID**判定每个事件是否属于本次 run——子会话（及孙会话）的 permission ask/question 归本 run，别的客户端（如 TUI）的会话不动。记忆化在 `owned` Map，`GLOBAL_FORM_SESSION_ID` 特判为不属于任何人。这与 TUI 的 discover 是同一判定（链上溯），只是一个 pull 一个 push。

---

## 9. 不变量与设计取舍

1. **子会话是一等 Session**：除 parentID 列外与顶层会话同构——同一 inbox/drain/投影/权限/工具管线。不引入第二套执行引擎。（AGENTS.md Session Core：SessionExecution 进程全局、按 Session ID；无 subagent 特例。）
2. **parentID 单写点**：只有 subagent 工具 execute 设（fork 走 fork_session_id 另一列）；无外键，运行时守卫兜底。=> 归属判定（CLI/TUI/深度守卫/恢复校验）全部退化为"沿链上溯"，无需专门索引查询。
3. **Job ID == childSessionID**：注册天然幂等（join 跳 run）；后台恢复凭据 recovery 四字段足以重建全部上下文。
4. **失败也带 sessionID**（subagent.ts:245-251 注释）：error/cancelled 的 ToolFailure 都内嵌 childID——子会话永远可续接（下次调用传 sessionID 即可），失败不等于丢弃。
5. **通知幂等双层**：synthetic 复用 notificationID 为 inbox 项 id（重投 reconcile 掉）+ notify 的 `childID:startedAt` key 去重（同代只投一次）。
6. **prompt 准入在 job 外**（subagent.ts:203-205 注释）：因为 Job.start join 运行中 job 会跳 run effect，而 prompt 的默认 wake 语义（idle 启动/运行中 steer）恰好正确。
7. **深度限运行时查**：schema 校验不了会话拓扑，execute 每次实数链；默认 1 层防递归爆炸，配 `experimental.subagent_depth` 放开。
8. **描述即权限视图**：hook 动态把 Permission.evaluate 非 deny 的 subagent 清单写进工具描述（:282-309）——模型可见性 = 可调用性。
9. **恢复不复活孤儿**：恢复前校验父子双存在、parentID 匹配；不匹配只清账不投递（restart.ts:143-147）——孤儿通知会唤醒无意义的父 drain。
10. **CLI 注释的原话**即归属不变量："Subagents run in child sessions; their asks and questions belong to this run too. Other sessions on a shared server must be left alone."（noninteractive.ts:140-141）。

---

## 10. 失败与恢复汇总

| 故障 | 表现 | 恢复 |
|---|---|---|
| 未知 agent / primary agent / 深度超限 / 权限拒 | ToolFailure（模型可见，可改正重试） | subagent.ts:130-151 |
| 续接非自己的孩子 | ToolFailure `is not a child` | :164-167 |
| model 参数非法/不可用/变体不存在 | ToolFailure 带可用清单 | :75-98 |
| 子会话执行出错 | 前台：ToolFailure 带 sessionID；后台：deliver error 文本 | :246-249 / completion:34-35 |
| 子被取消（父 onInterrupt / 用户） | 同上 cancelled | :250-251 / :36 |
| 子完成但无文本 | `NO_TEXT` 常量占位 | completion:8 |
| 前台等待中父 fiber 中断 | onInterrupt 砍子 + cancel job | :234-240 |
| 父 drain 中断遗留 streaming 工具 | settleStaleToolCalls 结算 aborted（带 childID） | runner/llm.ts:362-381 |
| 进程死亡，后台子未完 | KV 里 pendingBackground | recoverSubagent 校验后续跑或清账 | restart.ts:138-190 |
| 恢复重试预算耗尽 | notify RESUME_EXHAUSTED（error） | restart.ts:163-166 |
| 孤儿子会话（父先删/不匹配） | 通知只 completeBackground 清账；claim 被 releaseChildClaims 清 | restart.ts:143-147, :205 |
| 通知重投 | synthetic 幂等键 reconcile；notify set 去重 | completion:38 / job:31-33 |

---

## 演进注记

旧库 4.5 篇的 `task` 工具（gitee 旧基线）与现码 `subagent`（9_tool.md 已确认改名）的**语义深化**，远不止换名：

1. **从"一次性调用"到"可持续会话"**：task 时代输出即终局；现码 `sessionID` 入参让子会话成为**可多轮续接的对话实体**（含换 agent/model 的 switch 语义，:171-182），工具描述明说"pass back later to continue"。
2. **前台/后台双模**：`background` 入参 + Job 注册表接入（block/background/持久化恢复）——task 无后台概念。后台完成经 synthetic 通知回流，且输出文本里写死"禁止轮询"的行为约束。
3. **崩溃恢复闭环**：recovery 凭据持久化 + restart.ts 专属 recoverSubagent——旧 task 的执行随进程死亡即丢。
4. **深度守卫配置化**：experimental.subagent_depth（默认 1）取代任何旧的硬限制。
5. **权限与可见性联动**：description 动态追加"当前可用 subagent 清单"，与 Permission.evaluate 同步过滤。
6. **前端一等公民化**：TUI 专设 1141 行 tracker（family 视图/blocker 转发/hydrate 重连），CLI ownsSession 链式归属——子会话的 ask/question/事件流被完整纳入父 run 的 UX。

---

## 相关篇

- 前置：[4_session.md](4_session.md)（Session 聚合、inbox 准入、drain、synthetic）——subagent 三机制中两个的底座
- 前置：[9_tool.md](9_tool.md) §4.9（subagent 工具输入 schema、权限点、与 13 个 direct 工具的对照）
- 关联：[11_permission.md](11_permission.md)（permission.assert 与 agent 级 ruleset）
- 关联：[../5_timeline/6_tui-cli-prompt-chain.md](../5_timeline/6_tui-cli-prompt-chain.md)（TUI/CLI 事件消费全景；本篇 §8 只取切片）
- Job 注册表本体（block/background/wait/pendingBackground 完整语义）：[17_job.md](17_job.md)
