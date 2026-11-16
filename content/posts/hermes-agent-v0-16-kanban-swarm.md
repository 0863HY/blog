---
title: "Kanban Swarm：把多智能体协作写进一张任务图（Hermes Agent v0.15 → v0.16 实录）"
date: 2026-11-16T00:00:00+08:00
tags: ["hermes-agent", "multi-agent", "kanban", "orchestration"]
author: "随心6"
---

设想一个任务：三个 agent 从调研、架构、运维角度同时分析同一个「多区域容灾方案」。动手之前得回答四个问题——谁拆分、谁等谁、共享上下文写在哪、某个 worker 崩了从哪恢复。这套编排若跑在进程内的 subagent 树里，进程一退，拓扑、进度、半成品一起消失；若另写外部编排器，就要在 Kanban 之外再造一套状态存储和可见性。

Kanban Swarm v1 选的是第三条路：不新建调度器，把一次 fan-out / fan-in 写成既有任务板上的依赖图。它值得拆解的地方不是「多 agent 并行」，而是它把复杂度转移到了哪里，代价是什么。

## 一、板子缺的不是并行，是拓扑

既有 Kanban 内核已有卡、父子依赖、dispatcher、notifier、dashboard，缺的是三样表达：协作的固定拓扑、所有分支都能读写的共享上下文、进程之外的重启恢复点。

swarm 加了一层薄拓扑助手。引入提交是 `3ee7a5546d`（2026-05-18），首见于 `v2026.5.28`（2026-05-28，版本 0.15.0）。它不是 v0.16.0 才有的功能：v0.16.0（tag `v2026.6.5`）只是带着它发版，且两版之间该文件只差一行——synthesizer 的强制技能从 `avoid-ai-writing` 换成 `humanizer`。

**为什么重要**：薄助手意味着不必单独布点、监控、写迁移；版本号错位则会让工程判断整体错位——「原子建图」并非发布即有（见第四节）。

## 二、拓扑与轮次：四层，写死

```
root（规划卡，创建后立即 done，兼共享黑板）
 ├─ worker_1 (ready)  ← parents=[root]
 ├─ worker_2 (ready)  ← parents=[root]
 └─ ...
verifier    (todo)    ← parents=[全部 worker]
synthesizer (todo)    ← parents=[verifier]
```

状态断言直接来自仓库测试：root 为 `done`、workers 全为 `ready`、verifier 与 synthesizer 为 `todo`；verifier 的父集合等于全部 worker id，synthesizer 的父只有 verifier。

root「创建后立即完成」是必须的：父卡全 `done` 后，子卡才能提升，root 先完成，worker 才能开跑；它继续存在，充当审计锚点和黑板。把编排指令和编排产物压进同一张卡，省一张表。

整个过程只有一轮：一次 fan-out、一次门禁、一次汇总，没有循环，没有迭代重跑。**为什么重要**：判断任务适不适合 swarm，看它是不是标准的两段式（发散→收敛），而不是有几个 agent；写作—评审—重写的循环套进来，只会得到一张永远跑不完的图。

固定四层的代价很直接：不支持任意 DAG 形状。严格串行链路（调研→写作→审校→发布）套 swarm 只会凭空多一层无意义的并行，应改用 `kanban create` 加 `kanban link` 手工表达父子依赖。

## 三、黑板：为什么不引入第二个调度器

黑板是 root 卡下的普通评论行，靠前缀 `[swarm:blackboard] ` 区分，负载是 `{"key": ..., "value": ...}` 的 JSON，写入走 `add_comment`，一次写入同时落一行 `task_comments` 和一条 `commented` 事件。

读取时逐条过滤：无前缀的跳过（人类评论安全共存）、`json.loads` 失败跳过、`key` 为空或不是字符串的跳过；其余逐键覆盖，并记录每个键的胜出作者。

模块 docstring 说得直白：状态留在既有的 `task_comments` / `task_events` 行里，dashboard、notifier、slash 命令、dispatcher 都不用改造。**为什么重要**：这是零新增服务、零新增迁移的选择，不是一致性或性能上的选择；可观测面因此天然对齐——板子上看得见的，就是编排真实发生的。

代价有两条。

一致性是弱保证。`post_blackboard_update` 不做版本号、不做 CAS、不读旧值；两个 worker 并发写同一 key，由评论顺序决定胜者，先写者的值被静默覆盖（仍留在历史里，但不进 merge 结果）。排序更微妙：`list_comments` 用 `ORDER BY created_at ASC`，`created_at` 是整秒时间戳且无次级键；同文件的 `list_comments_after` 正因此改用 `ORDER BY id ASC`，文档串写明按 rowid 排序。同一秒内的多条写入，其相对顺序不保证等于插入顺序——读代码即可得，不必压测。

形态上是「追加」：每条更新写一行新评论，读取走全量遍历，源码里没有分页或增量读取的分支。**结论**：黑板适合笔记，不适合账本。计数器、锁、配额、预算、去重表这类需要读—改—写的状态不要放进去，那是在赌评论顺序；要放就放「谁做完了什么、结论是什么、证据在哪」。

## 四、建图原子化：一个被版本号掩盖的修复

官方文档与二手资料里常见「建图原子化，dispatcher 和 dashboard 只能看到完整拓扑」。这句话对 HEAD 成立（截至 2026-11），对 v0.16.0 不成立：该版本的 `create_swarm()` 顺序执行，每次 `create_task` / `complete_task` 各开独立事务，`grep -c write_txn` 的结果是 0。

原子化是后续修复：`0fe4d90223`（2026-07-31）把建图包进 `kb.write_txn`，root 改为以 `initial_status="blocked"` 创建；`af0a418666`（2026-08-10）新增 `_activate_root_inline()`，把 root 的 `blocked→done` 放进同一事务。

几个实现细节值得借鉴：`write_txn` 是 `BEGIN IMMEDIATE`；root 用 `blocked` 起步，因为它在允许的初始状态内、又被 `recompute_ready` 视为可提升，于是「建图中」的 root 不会被 dispatcher 派发——用状态机做互斥，而不是加锁；激活用 `WHERE id=? AND status='blocked'` 的条件更新翻转，`rowcount != 1` 抛错触发整图回滚。提交后才 `recompute_ready`（它自带 `BEGIN IMMEDIATE`，不能在写事务里跑），再触发钩子。收尾也不能直接调 `kb.complete_task`：那会开启自己的事务并触发工作区清理、失败计数清零等提交后副作用，而外层事务此时仍可能回滚。

测试验证了这一点：`test_create_swarm_graph_is_atomic_and_rolls_back_partial_build` 用第二个连接在第一次 `create_task` 后立刻查 `COUNT(*) FROM tasks`，断言为 0；第三次 `create_task` 抛异常后，两个连接都读到 0 行；钩子触发时 `in_transaction` 为 `False`。读者只可能看到两种状态：没有新 swarm，或完整拓扑。

边界要划清：原子性只覆盖建图这一次操作，之后的派发、执行、重试仍由 dispatcher 异步推进。v0.16.0 到 HEAD 的 swarm 模块差异是 `134 insertions(+), 113 deletions(-)`，不是小修——引用代码必须注明 tag，否则会把 2026-08 的行为安到 2026-06 的版本上。

顺带一个现成反例：官方文档示例写 `--workers researcher,architect,sre`，CLI 只有可重复的 `--worker PROFILE:TITLE[:SKILL,SKILL]`，照抄会直接报 `unrecognized arguments`。该错误在 v0.16.0 文档里就存在，截至 2026-11 线上仍未修正。**为什么重要**：拓扑助手是「一条命令建整张图」，命令错了这个功能等于不存在，文档就是它全部接口面。

## 五、什么时候不该用

两条结构性缺陷。

门禁是约定，不是校验：verifier 的指令只写在 prompt 正文，「complete only with metadata {"gate": "pass"} when evidence is sufficient」。代码里没有任何地方校验 metadata 是否真含 `gate=pass`；synthesizer 能否启动只取决于 verifier 卡是否 `done`。门禁强度等于提示词的强度，无法从系统层面断言「没过门禁的产物不会进入汇总」。

verifier 是串行收敛点。N 条并行分支全部汇到一张卡，而它的 block 是 sticky 的：`recompute_ready` 跳过它，只有显式 `unblock` 才能恢复。它一卡，整条链路停住且不会自愈。并行换来的收益上限，由收敛点上那一环决定——结构性论断，不是具体数字。

配置面还有几个缺口：CLI 的 `swarm` 子命令没有暴露 `max_runtime_seconds`、`max_retries`、`goal_mode`；worker 至少一个，代码不校验数量上限；并行度由 dispatcher 的 `kanban.max_in_progress` 决定，文档写明默认不设限；失败预算继承 `kanban.failure_limit`（默认连续 2 次 spawn 失败即自动 block），swarm 自己没有重试策略。

三类场景明确不该用：

- 严格串行链路：用 `kanban create` + `kanban link`，别为了「并行」硬塞两个 worker。
- 需要条件分支、动态 map-reduce 或循环重跑：Kanban 的依赖模型只认「父卡全 done 才提升」。这时外部编排器是对的，代价是你得重新解决状态可见性与重启恢复——那正是 swarm 帮你省掉的部分。
- 单个 agent 一次会话能做完的小任务：swarm 的账是「N 条并行分支」换「N+2 张卡（不含 root）+ 一次门禁 + 一次汇总」，任务不够大就是纯开销。

## 六、观点

swarm 的本质是拓扑 DSL 的语法糖，不是编排引擎。它的价值不在「多 agent」，而在把并行协作的恢复点落到一块你早就在看的板子上：卡是状态，评论是上下文，依赖是控制流，进程可以死。

选型只需问两个问题：瓶颈是不是那个收敛点？能不能接受一块弱一致、只有追加语义的黑板？都答「是」，它省下的相当可观；任一答「否」，你需要的是别的形状的图，而那张图得自己画。
