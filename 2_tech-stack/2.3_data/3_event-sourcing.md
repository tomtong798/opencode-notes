# 事件溯源（Event Sourcing）

## 是什么

架构模式：**不存"当前状态"，存"发生过的事实"**。

- 状态变更 = 向事件日志**追加**一条不可变事件（含 aggregate ID 与单调序号）
- 当前状态 = 按序回放（fold）事件推导
- 术语：**aggregate**（聚合根，如一个 Session）、**durable event**（持久事件/事实）、**replay**（回放）

## 业界类比

- 银行流水 vs 余额快照——存流水，余额随时可算、可对账、可审计
- 数据库 redo log 的"业务层放大版"

## 在 OpenCode 中的角色

- **core 的根本存储哲学**（规范：`specs/v2/session.md` + 根 AGENTS.md「Session Core」节）：
  - `event` + `event_sequence` 表是真相源：按 aggregate（Session ID）有序
  - 一切 Session 行为 = 发布 durable 事件（`session.inbox.enqueued`、`session.step.ended`…约 40 种，见 overview 附录 B）
  - **durable 事件最小化**红线：只记不可简的新事实，可由回放推导的状态不重复存
  - 事件回放（`sessions.log`：after/follow）与执行所有权分离；live 增量（文本 delta）不保证重放
- **write-ahead claim（执行占用标记）**：事件溯源在"进程生命周期"上的延伸——drain 开始先落 claim；成功/失败/用户中断释放，关机中断/进程死亡保留；启动恢复据此有界重试（不保证 exactly-once）
- inbox 投递的原子性：`session.inbox.delivered` 投影在同一事务内"删 inbox 行 + 插可见消息"

## 与其他技术的关系

- **CQRS 投影**：事件流如何变成可查表（见 [cqrs-projection.md](3.1_cqrs-projection.md)）
- **SQLite**：事件与读模型同库同事务（见 [sqlite.md](1_sqlite.md)）
- **Bus**：core 的事件总线，publish 即"追加事实 + 通知投影/订阅者"

## 相关源码

- `packages/core/src/bus.ts`、`event/sql.ts`
- `packages/core/src/session/projector.ts`、`execution.ts`（claim）、`inbox.ts`
- 规范：`specs/v2/session.md`（「Durable Events Are Session-Scoped」「Recovery Boundaries」节）
