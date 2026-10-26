---
title: "为什么多 Agent 编排不该只有图：从 v0.16 到 v0.21 的 Hermes Kanban Swarm 调度设计"
date: 2026-10-26
tags: ["Hermes Agent", "多 Agent", "Kanban", "AI 工程化", "调度系统", "失败恢复"]
author: Q
---

凌晨两点半，告警响了。你那条“抓资料 → 撰写 → 校验 → 发布”的多 Agent 流水线在第三跳崩了。你重启进程，框架很配合地从最后一个 checkpoint 恢复了——只是那一步之前的一次 `git push` 已经生效，恢复之后它又推了一遍。你花了一个小时清理分支，又花两个小时给每个节点补幂等键。

这不是“图不够强”的问题。LangGraph 的条件边、循环、子图和 time-travel 都很强；CrewAI 用几十行就能拿到带 checkpoint 的角色流水线；AutoGen 的对话式协作在探索型任务上几乎无可替代。但它们的默认假设是：**这次运行活着，你的进程就是世界的中心**。进程死了，世界停在最后一个快照上；operator 想插手，得等代码跑到那个 `interrupt()` 点；想换个执行者——对不起，执行者是 `def` 里写死的那个函数。

另一条路是把“任务”本身当成一等公民，丢进持久化队列，让 worker 变成可随时替换的消耗品。Hermes Agent 的 Kanban Swarm 走的就是这条：一张卡、一个状态机、一个外部 dispatcher、一个进程级 worker。这篇文章按这条主线拆它的调度内核——为什么抢卡只需要六行 SQL，为什么 review 是一个状态而不是一次回调，为什么“限流”和“心跳缺失”不计入失败预算。

如果你正在评估要不要把生产流水线搬过来，真正关心的多半是五个问题：状态存在哪、进程崩了怎么办、不同角色怎么隔离、人要怎么插手、能不能用一条命令复现。下面按这个顺序展开，每个机制尽量回答“为什么这么设计”，而不是“它长什么样”。最后给一套本机真实跑通的命令，以及一份“文档没写但你一定会踩”的清单。

## 先把版本说清楚

这件事必须先交代，否则你会拿着错的文档对不上号：

- `hermes kanban swarm` 这个拓扑助手随 **v0.15.0（2026-05-28，“The Velocity Release”）** 引入，PR #28443，salvage 自 #26791。Release notes 原文是“`hermes kanban swarm` creates a full Swarm v1 graph in one command”。来源：https://github.com/NousResearch/hermes-agent/releases/tag/v2026.5.28
- **v0.16.0（2026-06-05，“The Surface Release”）** 的 release notes 全文搜“swarm” 命中 0 次，那一版的主角是桌面 App。它对 Kanban 的增量是 `goal_mode`、任务附件、`default_assignee` 兜底与 per-profile 并发上限。来源：https://github.com/NousResearch/hermes-agent/releases/tag/v2026.6.5
- 持久化多 Agent Kanban 出现得更早，v2026.5.7（“The Tenacity Release”）的“Durable multi-agent Kanban with retries and recovery”。来源：https://hermes-ai.net/changelog/

所以标题里保留 v0.16 只作为时代锚点。准确的说法是“v0.16 时代的 Kanban Swarm”——它是既有能力，不是 v0.16 的发布物。本文所有机制细节以本机实测的 **v0.21.3（2026-09-14，upstream 8a92051f）** 源码与 CLI 输出为准；如果你按 v0.16 的参数去对，可能对不上。

这一节的成本很低，但它决定了整篇文章的可信度。技术写作里最贵的错误不是判断错，而是“版本错位”——读者照着一份不存在的功能说明去部署，会在十分钟内失去对你的信任。

## 全景架构：board / task / profile / dispatcher

先把四个对象摆平，它们的边界比名字重要。

| 对象 | 一句话职责 | 关键边界 |
| --- | --- | --- |
| **Board** | 一条独立队列，一个独立 SQLite 库 | 独立 DB、独立 `workspaces/` 与 `logs/`；跨 board 禁止建依赖 |
| **Task** | 一行卡片记录 | 单个 assignee、一个状态、一次尝试对应一行 run |
| **Profile** | 具名 agent 身份 | 独立配置/密钥/记忆/技能/状态库；**不**隔离文件系统 |
| **Dispatcher** | 长驻循环，抢卡与回收 | 默认跑在 gateway 进程内，60 秒一 tick |

Board 的解析顺序是四层：`--board <slug>` > `HERMES_KANBAN_BOARD` > `~/.hermes/kanban/current`（`boards switch` 写入的指针）> `default`。非默认板落在 `~/.hermes/kanban/boards/<slug>/kanban.db`。slug 的字符集被限制成小写字母数字加 `-`/`_`，1–64 字符且必须以字母数字开头——`/`、`.`、`..` 在 CLI 层就被拒，这是防路径穿越的第一道闸。

**为什么要这么硬**：board 隔离的是“事实来源”。worker 启动时，dispatcher 会把 `HERMES_KANBAN_DB`、`HERMES_KANBAN_BOARD`、`HERMES_KANBAN_WORKSPACES_ROOT` 三个变量钉死到当前板，每个 `kanban_*` 工具读的都是这套值。如果允许跨 board link，一个父卡的 `done` 就会触发另一个库的 `recompute_ready`——两个独立事务边界被一个删除操作缝在一起，崩溃时的修复策略立刻变成“猜”。

Tenant 是板内的软命名空间，官方定义写得很直白：“Tenants are a soft filter; boards are the hard isolation boundary.” 实现上只有 `task.tenant` 非空时才注入 `HERMES_TENANT`，新任务未指定时按父任务顺序继承第一个非空 tenant。隔离手法是 workspace 路径加 memory key 前缀，board、dispatcher、profile 定义全部共享。换句话说：**tenant 是给你记账用的，不是给你做安全边界用的**。

Profile 是执行者的身份。每个 profile 是一个独立的 Hermes home：`config.yaml`、`.env`、`SOUL.md`、skills、memories、sessions、`state.db` 全部独立。创建即得命令别名，`hermes profile create coder` 之后你就有 `coder chat`、`coder gateway start`，本质是 `hermes -p coder`。`--description` 这个参数服务的是 Kanban 路由——帮助原文是“Used by the kanban decomposer to route tasks based on role instead of profile name alone”。

Dispatcher 是唯一会“主动做事”的角色。它默认不独立运行：`kanban.dispatch_in_gateway: true`，循环跑在 gateway 进程里，`dispatch_interval_seconds: 60`。已废弃的 `hermes kanban daemon` 在 `--help` 里直接写着“DEPRECATED — dispatcher now runs in the gateway. Use `hermes gateway start`”。

一个常被追问的设计细节：task 只有一个 assignee。为什么不允许“多人协作一张卡”——因为一旦一张卡有多个执行者，`claim_lock` 就失去意义，状态机退化成“谁先改谁赢”。需要并行就建多张卡，用一张 root 卡做聚合。swarm 拓扑的全部秘密就在这句话里。

这个设计的取舍值得说清：好处是零运维（不用装 systemd 服务）且调度与通知同进程（`notify+wake` 的唤醒语义只在活着的 gateway 里生效）；代价是**没有 gateway 就没有调度**，`ready` 卡原地不动，`hermes kanban create` 会打警告。多 profile 部署时官方要求只有一个 gateway 保持 `dispatch_in_gateway: true`——两个 dispatcher 打同一个 `kanban.db` 会造成 claim 竞争，明确标注 not supported。

### 一次派发的完整往返

抢到卡之后，dispatcher 要做的事比“起个进程”多。它得把 worker 变成一个坐标明确的进程：知道自己是谁、在哪张板、哪个目录、第几次尝试。v0.21.3 的注入清单（`kanban_db_dispatch.py:2769-2831`）大致是：

```bash
HERMES_HOME=<profile home>
HERMES_PROFILE=<profile 名>
HERMES_KANBAN_TASK=<task id>            # 身份键
HERMES_KANBAN_WORKSPACE=<workspace>     # 工作目录
TERMINAL_CWD=<workspace>                # 绝对且存在时才写
HERMES_SESSION_SOURCE=kanban
HERMES_KANBAN_RUN_ID / _CLAIM_LOCK / _GOAL_MODE / _GOAL_MAX_TURNS
HERMES_KANBAN_DB / _WORKSPACES_ROOT / _BOARD / _BRANCH
HERMES_TENANT=<tenant>                  # 仅 tenant 非空
```

最后一步是 `env.pop(DELEGATED_CHILD_ENV_MARKER)`，源码注释写的是“This is the grant boundary: the dispatcher assigned this new worker's task.” 换句话说，**移除这个标记本身就是授权动作**——不是“我允许你写”，而是“这张卡确实派给了你”。

其中 `TERMINAL_CWD` 最容易被当成冗余，它存在的理由写在代码注释里：不设的话，文件工具的相对路径会落到 gateway 用户的家目录，而且 worker 会去加载 gateway 的 `AGENTS.md`（#41312 / #34619）。这两件事都不会报错，只会让 worker 在错误的地方安静地工作——多 Agent 系统里最难查的一类 bug。

本节与下几节的事实来源：官方 Kanban 参考页 https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban、Profile 说明 https://hermes-agent.nousresearch.com/docs/user-guide/profiles、环境变量参考 https://hermes-agent.nousresearch.com/docs/reference/environment-variables，以及本机 v0.21.3 源码（行号已在文中标注）。

## 任务生命周期：九个状态和一个容易漏掉的

源码里的权威定义在 `kanban_db.py:103`：

```python
VALID_STATUSES = {
    "triage", "todo", "scheduled", "ready",
    "running", "blocked", "review", "done", "archived",
}
```

**注意官方文档的 Core concepts 一节只列了八个，漏了 `scheduled`**。`hermes kanban schedule <id> --at <ISO8601>` 落的正是这个状态——dispatcher 会跳过 `scheduled_at` 还在未来的卡，到点后的第一个 tick 取走。如果你要引文档原文，记得标注这个差异；如果你要写代码判断状态全集，以源码为准。

九个状态里，真正需要理解的只有四条路径：

- `triage → todo`：`hermes kanban specify` 用辅助 LLM 把一句话补成完整 spec；`decompose` 把它拆成子图。
- `todo → ready`：依赖满足。这是唯一由 dispatcher 批量推动的迁移。
- `ready → running`：CAS 抢卡成功。
- `running → {done, review, blocked}`：三种终局，语义完全不同。

另外有一个容易被忽略的结构：**task 和 run 是两张表**。一个尝试了三次的任务有三行 `task_runs`，`tasks.current_run_id` 指向当前那次。所以“这张卡历史上被谁跑过、每次因为什么失败”是可查的，而不是覆盖式的。这条设计直接支撑了后面所有恢复逻辑。对应的入口是 `hermes kanban runs <id>`，每次尝试一行，带 outcome、耗时和错误。

`scheduled` 这个状态值得单独说一句：它闸的不是人，是**时间**。`scheduled_at` 在未来的卡不会被 spawn，到点后第一个 tick 才进 lane。用途很具体——夜间批处理、跟着上游数据发布节奏跑的卡，以及“现在别跑，等配额重置”这种临时处置。它和 `todo` 的区别在于等待对象：`todo` 在等别的卡，`scheduled` 在等时钟。

`triage` 则是“还没想清楚”的容器。`create --triage` 落在这里，之后有两条加工路径：`hermes kanban specify` 用辅助 LLM 把一句话补成带验收标准的完整 spec 并 promote 到 `todo`；`hermes kanban decompose` 把它拆成子图，原子提交。`kanban.auto_decompose` 默认 true，但每 tick 最多处理 `auto_decompose_per_tick`（默认 3）张。为什么要有这个上限：批量灌 50 张 triage 卡，如果每 tick 全拆，就是一次几十个辅助模型调用的突发，账单和限流会同时到。

### 三条非显然规则

这三条是“读过源码”和“读过 README”的分水岭。

**第一，`blocked` 有两种性质。** 显式 `kanban_block` 是 sticky 的——只有 `kanban_unblock` 能让它退出，`recompute_ready` 会主动跳过它。但熔断器触发的 block（连续失败到 `gave_up`）只要父卡都完成、且失败数没到上限，可以被自动提升回 `ready`。同一个状态名，两种生命周期。

**第二，`kind=dependency` 会被改写。** 如果一个 worker 声称“我在等别的任务”，但实际没有未完成的父卡，`block_task` 会把它 re-kind 成 `needs_input`，payload 里记 `requested_kind: dependency` 和 `rekind_reason: no_open_parent`。为什么重要：不这么改，就会出现“blocked → 自动提升 → 无上下文重生 → 再 block”的死循环，每轮烧一次模型钱。

**第三，review 线在重试中不会丢。** `_retry_status_for_run()` 读该 run 的 `claimed` 事件里的 `source_status`，如果是 review 就退回 `review`，否则退回 `ready`。源码注释原话：“so crash/timeout/reclaim can't silently turn a reviewer run into an implementation run.” 这条的价值在于：审查者和实现者是两个不同 profile、不同技能集、不同权限假设，一旦串线，一次崩溃就会让“待审”变成“重新实现”。

### 安全 lane

有一个贯穿全文的词值得先定义：**安全 lane**。当一次执行以非正常方式结束时（崩溃、超时、被回收、stale），卡不是随便回 `ready`，而是回到“它本来该在的那条 lane”——实现线回 `ready`，审查线回 `review`。上面第三条规则就是这个原则的具体实现。

这个细节很小，但它保证了“恢复”不会改变任务的性质。一旦允许串线，一次 OOM 就能让一张待审的卡变成一张待实现的卡，reviewer 的历史被 implementer 覆盖，审计链断在一个你永远不会去看的地方。

### unblock 循环熔断

同一原因反复 unblock → 再 block，达到 `BLOCK_RECURRENCE_LIMIT = 2`（`kanban_db.py:111`）时，卡不再进 `blocked`，而是升到 `triage` 等编排介入，写 `block_loop_detected`。计数器只在 `complete` 成功后清零。

这个设计针对的是一类非常真实的多 Agent 故障：一个卡因为“缺 API key”被 block，人补了 key 解封，worker 起跑后发现是另一个 key 缺，又 block——人类的处理动作变成了系统的一部分，循环会一直转下去。熔断把它强制升级成“需要重新设计这张卡”，而不是“再试一次”。

## dispatcher：抢卡、回收与依赖 promote

### 抢卡：六行 SQL 代替分布式锁

并发模型写在 `kanban_db.py:7-9` 的模块 docstring 里：

> Concurrency: WAL + `BEGIN IMMEDIATE` + compare-and-swap on `tasks.status`/`claim_lock` — SQLite serializes writers so one claimer wins, losers see zero rows (no retries, no distributed locks).

CAS 本体（`kanban_db.py:2221-2235`）：

```sql
UPDATE tasks
   SET status = 'running',
       claim_lock = ?,
       claim_expires = ?,
       started_at = COALESCE(started_at, ?)
 WHERE id = ?
   AND status = 'ready'
   AND claim_lock IS NULL
```

`cur.rowcount != 1` 就返回 `None`，**不重试**。同一个写事务里插 `task_runs` 行、更新 `current_run_id`、写 `claimed` 事件——所以“认领成功但 run 没开”这个中间态在物理上不存在。

**为什么这个设计值得抄**：它把“谁拿到任务”这个分布式共识问题，降级成“谁能写进这一行”。SQLite 的 `BEGIN IMMEDIATE` 保证同一时刻只有一个写事务；`status='ready' AND claim_lock IS NULL` 是条件，不是检查（check-then-act 的经典竞态在这里被消掉了）。代价是**单机**：崩溃检测假设 PID 是 host-local，跨主机共享一个 board 官方明确不支持。这是一次有意识的取舍——用“不支持”换“不需要”。

TTL 默认 15 分钟（`DEFAULT_CLAIM_TTL_SECONDS = 15 * 60`，`kanban_db.py:276`），可用 `HERMES_KANBAN_CLAIM_TTL_SECONDS` 覆盖。worker 正常工具活动大约每分钟自动续期一次，续期本身也是条件更新：

```sql
UPDATE tasks SET claim_expires = ?
 WHERE id = ? AND status = 'running' AND claim_lock = ?
```

`rowcount != 1` 就返回 False，语义是“我们已经不拥有它了”。这是带条件的续约：卡被 reclaim 后重新派给别人时，老 worker 的心跳不会覆盖新 claim。

还有一道双保险：`claim_task()` 在 CAS 之前先查父卡，父卡没 done 就把卡 `ready → todo` 并记 `claim_rejected{reason: parents_not_done}`（`kanban_db.py:2278-2284`）。CAS 失败只说明“这一行被人动过”，父依赖检查说明“这一行本来就不该被抢”。

### priority 只是 tie-break，不是抢占

取卡 SQL（`kanban_db_dispatch.py:2211-2217`）：

```sql
SELECT id, assignee FROM tasks
 WHERE status = '{status}' AND claim_lock IS NULL
 ORDER BY priority DESC, created_at ASC
```

数值越大越先，同优先级先建先跑（FIFO）。列表和看板列用同一套排序，所以“看板最上面那张就是下一个被 spawn 的”。

CLI 帮助里的措辞是“Priority tiebreaker”——注意它不抢占已在跑的卡。很多调度系统会把 priority 做成可抢占式，这里刻意没做：抢占意味着“中止一个正在进行的 LLM 会话”，而会话的副作用（已写的文件、已发的请求）不可回滚。宁可慢一步，不要多一个不确定状态。

### promote：每个 tick 扫一遍，不是事件驱动

`recompute_ready()`（`kanban_db.py:2127-2186`）扫 `todo` 和 `blocked`，**所有**父卡状态属于 `{done, archived}` 才提升。目标状态由 `_resume_status_from_events()` 决定——review 线的卡回 `review`，实现线回 `ready`——并写 `promoted` 事件。

触发点是 `kanban_db_dispatch.py:2147` 的 `result.promoted = _kb.recompute_ready(...)`，也就是说：**父卡完成到子卡可跑，延迟上限是一个 tick（默认 60 秒）**。

为什么不事件驱动？因为事件驱动要求“父卡 complete 的那一刻，链路是通的”。如果父卡是在 CLI 里手工完成的、或者 dashboard 上点的、或者父卡归档触发的——三条路径都要各自记得通知下游，漏一条就永久卡死。轮询的代价是 60 秒延迟，收益是“任何能让父卡变成 done 的操作都会自动生效”。

### link 的三道硬拒绝

事后加边 `kanban_link(parent, child)` 的行为值得单列，因为它暴露了设计的边界：

```
kanban: cannot link t_A -> t_B: child is already running
kanban: linking t_B -> t_A would create a cycle
kanban: a task cannot depend on itself
```

如果 child 已经是 `ready`，加边会把它**降回 `todo`**，返回 `gated: true` 并记 `dependency_wait{demoted: true}`。这是对的——你刚声明了一个依赖，它当然不该继续可跑。

但 child 已经 `running` 时直接拒绝，理由是：抢卡之后再加边，**无法把已经开始的执行串行化**。这个拒绝比“默默允许”诚实得多。

由此派生出官方警告的经典死锁：**不要把“支持卡”link 到它要解锁的父卡**。那张 support 卡会变成父卡的 child，被父卡 gate 住，父卡又在等 support 卡的产出——双双饿死。正确做法是在 support 卡的 body 里写父卡 id，让关系停留在文本层面而不是依赖层面。唯一解封出口是 `hermes kanban unlink <parent> <child>`。

### 失败语义：退出码就是分类器

这是整个系统里最细的一块，也是最值得别的编排框架抄的。

| 退出码 | 含义 | 系统反应 |
| --- | --- | --- |
| `1` | 普通失败 | 计入 `consecutive_failures` |
| `75`（EX_TEMPFAIL） | 限流 / 5xx / 配额墙 | 记 `rate_limited`，重排队，**不计失败** |
| `78`（EX_CONFIG） | 凭证或模型被拒 | 首跳即 sticky `blocked`（`terminal_provider: true`） |

**为什么 75 要特殊对待**：配额墙不是卡片的错。把 429 计成失败，一张卡在早高峰跑两次就自动 block 了，而它其实一行代码都没问题。这里的设计判断是“区分环境故障与任务故障”，环境故障只退避，不消耗重试预算。

冷却由 respawn guard 负责：`rate_limit_cooldown` 和 `infrastructure_cooldown` 共用 300 秒窗口，只“按住”，永不计数。此外还有 `blocker_auth`、`recent_success`、`active_pr` 三种守卫，专防 worker 风暴——同一张卡在一个 tick 里被反复 spawn，是这类系统最典型的资源事故。

### 超时与 stale：不怪 worker

`dispatch_stale_timeout_seconds` 默认 14400（4 小时），配合 `_STALE_HEARTBEAT_GAP_SECONDS = 3600`（1 小时）判定：**跑超 4 小时且最近 1 小时没有心跳**才判 stale。命中后 SIGTERM 本机 worker，卡回安全 lane，run 以 `outcome='stale'` 关闭，**不计失败**。源码理由写得很克制：“dispatcher-side absence detection, not a worker fault.”

还有一个容易被忽略的保护：如果 SIGTERM 之后 worker 仍然活着，**不释放 claim**，而是下一 tick 重试并记 `_defer_reclaim_for_live_worker(reason="heartbeat_stale_worker_alive")`。因为一旦释放，dispatcher 会立刻在旁边 spawn 一个新的——两个进程同时写同一个 workspace，比慢一点糟得多。

熔断阈值 `kanban.failure_limit` 默认 2，解析顺序是任务 `max_retries` > `kanban.failure_limit` > 内置默认。协议违约（worker 退出码 0 但卡还 `running`，典型是模型只输出了一段文本、没调用任何终态工具）单独统计，上限 3，且只算**连续**值——中间的 rate_limited 重排队是中性的，其他失败会把连击清零。

`max_in_progress` 未设时的默认值也值得注意：不是“无限”，而是按 `MemTotal/512MiB` 推算并 clamp 到 `[2, 8]`；读不到内存的 macOS / Windows 才真的不限制。这是个很务实的兜底——防止有人在 8GB 笔记本上一次派 40 个 worker。

### swarm 只是往看板上写了一张图

`hermes_cli/kanban_swarm.py` 的模块 docstring 自述：

```
planning root (completed immediately)
    ├─ parallel specialist workers (ready)
    └─ verifier (todo until all workers done)
         └─ synthesizer (todo until verifier done)
```

设计原则同一处原文：**“Deliberately no second scheduler — a small task graph written into the existing Kanban kernel.”**

实现上 `create_swarm()` 在**一个 `write_txn`** 内建完 root（`initial_status="blocked"`）→ N 个 worker（`parents=[root]`）→ verifier（`parents=worker_ids`，自动挂 `requesting-code-review` skill）→ synthesizer（`parents=[verifier]`），随后把 root 翻成 `done` 并 `recompute_ready()`。所以读者只会看到两种状态：**完全没有这个 swarm**，或者**完整拓扑**。半个图是不可观测的。

共享黑板是 root 卡上的结构化 JSON comment，前缀 `[swarm:blackboard] `。这一手很聪明：dashboard、notifier、slash command、dispatcher 全部零改动继续工作，因为对它们来说那只是一条普通评论。门控完全复用 `todo → ready` 的提升逻辑，没有第二套 gate。

## profile 隔离与 workspace 三形态

### profile 隔离状态，不隔离文件系统

这是最容易被写错的一点。每个 profile 有独立的配置、密钥、人格、记忆、会话、技能、定时任务、状态库——**但它不提供沙箱**。官方文档的原文是“Profiles do **not** sandbox the agent”；local 后端下 agent 的文件权限等同于你的用户账号，profile 拦不住它访问 profile 目录之外。

所以三个概念正交：**profile 是状态目录，workspace 是工作目录（由 `terminal.cwd` 控制），sandbox 才限制文件系统访问**。把 profile 当沙箱用，是这类系统最常见的误解。

反模式也随之明确：绝不要让两个 agent 进程指向同一个 profile。两边都会自动写 memory 并互相加载，状态会互相污染。

### 工具集门控：worker 拿不到全量

Kanban 工具只在两种条件下注册：dispatcher 派发的 worker（`HERMES_KANBAN_TASK` 已设），或该 profile 显式启用了 `kanban` toolset。而且两档门控给的东西不同——`tools/kanban_tools.py:93-102` 的注释原文：“Board-routing tools (kanban_list, kanban_unblock): hidden from task workers.”

为什么重要：一个 worker 如果能 `kanban_unblock`，它就能解开自己的卡并无限重生，熔断器形同废纸。能 `kanban_list` 就能看见别的任务，然后试图越界。最小权限在这里不是安全洁癖，是让状态机的不变量成立。

更狠的一道在 shell 层。worker 的 bash 子进程**看不到** `HERMES_KANBAN_TASK`——`scrub_kanban_env()` 把身份键（task / run_id / claim_lock / goal 参数）剥离，改写为一个 fence marker（值为 board root），`kanban_path_is_fenced()` 据此拒绝对被钉住的库、或 fence root 下任何库的写操作。源码注释点明了动机：“TASK absence alone would promote a descendant to an orchestrator.”

结论很实用：**bash 子进程拿不到 board 写权限，写板只能走 agent 进程内的 `kanban_*` 工具**。所以别在 worker 里写 `hermes kanban complete $ID` 这种脚本——它会在 CLI 层快速失败并报“delegate_task child contexts cannot mutate Kanban tasks via the CLI”。

### workspace 三形态决定产物生死

| 形态 | 路径 | 完成后的命运 |
| --- | --- | --- |
| `scratch`（默认） | `boards/<slug>/workspaces/<task_id>` | **删除**，除非显式声明 artifacts |
| `dir:<绝对路径>` | 你指定的已有目录 | 保留 |
| `worktree` | 仓库 `.worktrees/<task_id>` | 保留，清理有安全闸门 |

`dir` 形态要求绝对路径，相对路径在解析时直接报错，理由是“相对于 dispatcher 当时的工作目录解析会产生混淆代理式逃逸”。

`worktree` 在任务未指定路径时锚定 board 的 `default_workdir`，落在 `<repo>/.worktrees/<id>`；**board 没设默认目录、或不在 git 仓库内，直接报错而不猜**。

清理闸门有三道，都和“别删错东西”有关：

1. scratch 只删受管路径——必须严格位于 `HERMES_KANBAN_WORKSPACES_ROOT`（或默认的 `kanban/workspaces`、`kanban/boards/<slug>/workspaces`）之下，否则**拒绝 rmtree** 并告警。这条是防“board 设了 default_workdir + 用了 scratch”组合误删用户源码树。
2. 存在未完成子任务时**延后**清理——子任务还要读父卡的 handoff 产物。
3. worktree 只在“工作树干净 + 每个 commit 都能从远程跟踪分支到达”时才 `git worktree remove`，且**不加 `--force`**，让 git 二次校验。

CLI 上的取值就三种：`--workspace scratch|worktree|worktree:<path>|dir:<path>`。帮助里有一句容易忽略的话——“an explicit 'scratch' also opts out of a project-scoped board's project”。也就是说，显式写 `scratch` 不只是选一个目录，它还让你脱离 project 绑定。这个副作用值得记住，因为字面意思和实际行为差得有点远。

默认选 `scratch` 的取舍也该讲清：绝大多数 agent 任务都是“产出几个文件、把结论写进 summary”，这些产物本该走 artifacts 通道而不是留在目录里。用一个完成即删的目录，等于强制你把“什么是要交付的”显式声明出来——顺手治好了“产物散落一地、没人知道哪个是最终版”这个毛病。

## 产物如何跨卡片传递

多 Agent 流水线真正难的不是并行，是“下游怎么拿到上游的东西”。这里有三条通道，各有明确用途。

**第一条：`summary` + `metadata`。** `kanban_complete` 的 summary 是 1–3 句人类可读交接，metadata 是自由 dict。下游 `kanban_show()` 会在 `Parent task results` 段落里看到全部父卡的 handoff，带创建时间。注意这些是**时点快照**，工具说明里明确写了“not live state”——如果你的结论依赖父卡的某个结果、而它已经过去很久，应该重新核实。

**第二条：评论线程。** `kanban_comment` 写的是永久行，worker 重派时会读到完整历史。这就是为什么“恢复”在这套系统里不是 replay 状态，而是“re-spawn + 重读注释线程”——新 worker 拿到的是上下文，不是内存镜像。它还有个隐式价值：这是唯一可以被人随时写、又不会打断 agent 的通道。worker 在跑时你写一条，它下次读上下文会看到；如果它已经跑完，这条评论就变成下一次交接的前置信息。这比“给 agent 发消息”更可靠，因为它落在库里，不依赖任何进程活着。

**第三条：附件与 artifacts。** 这是唯一能带走真实文件的路。`kanban_complete(artifacts=[...])` 和 `kanban_request_review(artifacts=[...])` 会把声明的文件先拷进持久附件存储（`boards/<slug>/attachments/<task_id>/`），然后才允许 scratch 清理。两个细节值得记住：

- **声明了但文件不存在的 artifact，会让任务卡在 in-flight**，以便你改路径重试——不会静默丢掉。
- 附件上限 25 MB，超了 CLI 直接拦：`kanban: attachment exceeds 25 MB limit`。

还有一条容易混淆的通道：`kanban_attach` / `kanban_attach_url` 是**给人**的附件，它挂在任务上供下载、供看板展示，不参与 handoff 语义；`artifacts` 是**给下游**的，它参与持久化与清理的顺序。混用之后的表现是“人在看板上下到了文件，但下游 worker 说没收到”。

跨板也是同一个逻辑：`artifacts` 里的绝对路径可以指向任何位置，但下游拿到的持久副本在 `boards/<slug>/attachments/<task_id>/`。所以当 scratch 被清掉、原路径失效时，附件还在。反过来，如果你只把产物路径塞进 metadata 而不走 artifacts，一次 scratch 清理就等于把交接链剪断了——而报错要等到下游读文件时才会出现，那时已经隔了好几张卡。

**这条链的工程含义**：handoff 是有类型的。`summary` 给人看，`metadata` 给机器读，`artifacts` 给下游当输入，`comment` 给审计。绝大多数多 Agent 事故不是模型不够聪明，是这四样混在一起——把一段散文式的总结当结构化数据用，下游解析失败，然后开始猜。

实际写交接契约时，metadata 的字段名最好从第一天就固定下来。一个够用的最小集是：

```json
{
  "changed_files": ["src/foo.py"],
  "tests_run": 12,
  "artifact_path": "/abs/path/report.md",
  "decisions": ["选 SQLite 而非 Postgres，因为单机单用户"]
}
```

`decisions` 这一项经常被略掉，但它恰恰是下游最需要的：下游能读到你改了哪些文件，读不到你为什么这么选。把它写进 metadata，下一个 worker 就不必把同一段推理再做一遍，也不会做出与你相反的选择。这条经验在任何编排框架里都成立，只是 Kanban 把它变成了显式字段。

## 横向对比：LangGraph / CrewAI / AutoGen

四者不在同一抽象层，这是所有对比的起点。Kanban Swarm 是“持久化工作队列 + 进程级 worker + 外部 dispatcher”的调度系统；LangGraph、CrewAI、AutoGen 是“进程内多 Agent 编排库”。分水岭是**任务状态归谁所有**——前者放进 SQLite，由独立进程轮询驱动；后三者放在你的 Python 进程里（可选落盘快照），由你的代码调用驱动。

| 维度 | Kanban Swarm | LangGraph | CrewAI | AutoGen |
| --- | --- | --- | --- | --- |
| 定位 | 持久化 SQLite 队列 + 进程级 spawn | 图状态机（节点+边+checkpoint） | 角色扮演 + 顺序/层级任务流 | 会话式多 Agent 对话编排 |
| 状态持久化 | 队列状态本身（DB 行）；恢复=重派+重读注释 | checkpointer 存图快照 | Flow `@persist`（SQLite）+ Crew `checkpoint` | 手动 `save_state`/`load_state` |
| 崩溃后能否续跑 | 能：回收→回 ready；4h 且 1h 无心跳判 stale | 取决于 durability 档位 | 能，`from_checkpoint()` 跳过已完成 task | 能，但存什么、何时存自己定 |
| 依赖/DAG | 显式 parents + promote | 任意有向图（条件边/循环/子图） | Crew 顺序或层级；Flow `@listen` | 无 DAG：轮转/选择器/handoff |
| 隔离粒度 | 每 worker 一个 OS 进程（单机单用户） | 同进程 | 同进程 | 默认同进程；gRPC 分布式（实验性） |
| 失败重试 | 退出码分类 + 熔断 + respawn guard | 需自建；checkpoint 提供恢复点 | `guardrail_max_retries`（默认 3）+ checkpoint | 终止条件 + 自建重试 |
| 人工审查门 | 一等状态：`review` 列 / block 分类 / 随时介入 | `interrupt()` + `Command(resume)` | `human_input=True` + guardrail | `UserProxyAgent` / run 间反馈 |
| 最小使用成本 | 需常驻 gateway/dispatcher | 需自运维 checkpointer 后端 | 最低，几十行拿到流水线 | 中，持久化要自己接 |

几条具体差异值得展开：

**持久化的粒度不同。** LangGraph 的 checkpointer 逐“超级步”写快照，durability 三档：`exit`（只在退出/报错/interrupt 时写，无中途恢复）、`async`（与下一步并行写，崩溃有小概率丢）、`sync`（每步前写完，最稳但慢）。`MemorySaver` 重启即丢，生产要 Postgres 或 SqliteSaver。更要紧的是 `interrupt()` 恢复时**所在节点从头重跑**，interrupt 之前的副作用会重放——幂等必须你自己保证。这正好对上开头那个凌晨两点半的故事。

CrewAI 两层都有：Flow 的 `@persist`（默认 `SQLiteFlowPersistence`）支持 resume 与 fork；Crew 的 `checkpoint=True` 在每个 task 完成后落盘。粒度是任务级，比 Kanban 细一档。AutoGen 最自由也最费事：`save_state()` / `load_state()` 是手动的，时点、内容全由你决定，框架不自带自动 checkpoint。

**“失败”的定义不同。** Kanban 里失败是分类型的（1/75/78），有预算（failure_limit），有冷却（respawn guard）；后三者普遍只有“异常抛出”和“重试次数”。差别在结果上很直观：Kanban 里“某个 agent 卡住”是一个运维事件，需要 `diagnostics` 去看；LangGraph 里它是程序异常，栈一打印就知道。

**人工门的形态不同。** 三者都是“代码里的一个点”：LangGraph 的 `interrupt()`、CrewAI 的 `human_input=True`、AutoGen 的 `UserProxyAgent`。Kanban 的门是“队列里的一个状态”——agent 还没起跑时，人就能改 assignee、改 priority、加一条 comment、或者干脆 unlink 掉一个依赖。区别在于谁来遵守时序：前者的时序写在代码里，后者的时序写在数据里。

**隔离的层级不同。** 同进程意味着一个未捕获异常或 OOM 会带走整个编排。AutoGen 另有 gRPC 的 Distributed Agent Runtime 可跨进程跨语言，但官方标注 experimental。需要坦白的是：LangGraph Agent Server 在多进程/多租户下的隔离细节，本节未核实；AutoGen 分布式 runtime 的生产成熟度同样只有 experimental 标注，没有稳定性承诺。

**怎么选。** 任务跨小时跨天、需要审计轨迹、不同角色要用不同模型和密钥、崩溃必须续跑、人要能随时插话——选 Kanban Swarm，放弃的是任意控制流（只有 DAG + 状态机）。需要细粒度条件分支与 time-travel，且愿意自己运维 checkpointer——选 LangGraph。角色化顺序流水线、想用最少代码拿到任务级 checkpoint——选 CrewAI。以对话涌现协作为核心的探索型任务——选 AutoGen。

顺带说清同一仓库里的另一个原语：`delegate_task` 是 RPC 式 fork→join，父等子返回、子匿名、失败即失败、无 HITL、审计随上下文压缩丢失。Kanban 是持久化消息队列加状态机：create 之后 fire-and-forget，子是有持久记忆的具名 profile，可 block → unblock → rerun，一张卡生命周期内能换 N 个 agent。一句话：`delegate_task` 是一次函数调用，Kanban 是一个工作流。

本节框架侧结论的来源（均为文档核实，未做本机实测）：LangGraph 持久化 https://docs.langchain.com/oss/python/langgraph/persistence、interrupt 语义 https://docs.langchain.com/oss/python/langgraph/interrupts、durability 三档 https://reference.langchain.com/python/langgraph/types/Durability；CrewAI Flows https://docs.crewai.com/en/concepts/flows、Checkpointing https://docs.crewai.com/v1.15.2/en/concepts/checkpointing、Tasks（`human_input` / guardrail）https://docs.crewai.com/en/concepts/tasks；AutoGen Teams https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/teams.html、State https://microsoft.github.io/autogen/stable/user-guide/agentchat-user-guide/tutorial/state.html、Distributed Agent Runtime https://microsoft.github.io/autogen/stable/user-guide/core-user-guide/framework/distributed-agent-runtime.html。

## 最小可复现示例：一条流水线从零到发布

下面这套在本机 v0.21.3 上真实跑通过。**第一件事是建独立实验板，永远别在生产板上做实验。**

需要如实说明的是：每一步都有真实输出，但全程**没有真实 spawn 过一个 worker**——派发环节用 `claim` 和 `--dry-run` 代替，避免与并行的调研卡抢模型配额。所以“事件流对不对”是实测的，“worker 被拉起来之后内部长什么样”这一点本文没有直接观察。

```bash
# 0) 独立实验板（独立 SQLite，独立 workspaces/ 与 logs/）
hermes kanban boards create kanban-demo --name "Kanban 演示板"
hermes kanban --board kanban-demo init      # 幂等，会打印可用 profile 名单

# 1) 建父卡并立刻 claim（避免 60s tick 把卡派给真 worker，白花模型钱）
P=$(hermes kanban --board kanban-demo create "父卡：采集数据" \
      --assignee researcher --body "产出实测日志" --json | jq -r .id)
hermes kanban --board kanban-demo claim $P

# 2) 建子卡并挂依赖：自动落 todo，等父卡 done
C=$(hermes kanban --board kanban-demo create "子卡：汇总成稿" \
      --assignee writer --parent $P --json | jq -r .id)

# 3) 三个视角各看一遍，缺一不可
hermes kanban --board kanban-demo list        # 看板视角：状态 + assignee
hermes kanban --board kanban-demo show $P     # 单卡视角：events + runs + children
hermes kanban --board kanban-demo context $C  # worker 视角：worker 真正看到的上下文

# 4) 完成父卡（带结构化交接），子卡在下一个 tick 自动 promote
hermes kanban --board kanban-demo complete $P \
  --summary "父卡实测完成" --metadata '{"tests_run":3}'

# 5) 派发自检：--dry-run 是零成本的安全网
hermes kanban --board kanban-demo dispatch --dry-run --json
hermes kanban --board kanban-demo dispatch --max 1

# 6) 事后补依赖、看历史、追事件流
hermes kanban --board kanban-demo link $P $C
hermes kanban --board kanban-demo runs $P          # 每次尝试一行
hermes kanban --board kanban-demo log $P --tail 400
hermes kanban --board kanban-demo tail $P
hermes kanban --board kanban-demo diagnostics      # 卡住原因
```

子卡的事件流长这样，建议直接当插图看：

```
created → dependency_wait{reason: parent_not_done} → promoted → claimed → completed
```

`promoted` 那条事件是 dispatcher 写的，`run_id` 为 NULL——因为它不属于任何一次 worker 运行。这个细节能帮你区分“系统推进”和“worker 推进”。

几个命令选择值得解释。`--json | jq -r .id` 不是为了好看：从 stdout 里 grep id，会在格式变化时静默取到空值，然后你会在 `claim ''` 上浪费十分钟。先 `claim` 父卡再建子卡，是因为 tick 间隔 60 秒——建完父卡就去改参数，很可能一轮派发已经把它交给真 worker，一次真实模型调用就这么发生了。`context $C` 是三视角里最有价值的一条：它打印的是 **worker 真正会看到的上下文**，包括父卡 handoff、评论线程、注入的指导语。看一眼就知道“我以为它知道的”和“它实际知道的”差多少。

### 一条命令起 swarm

```bash
hermes kanban --board kanban-demo swarm "多角度评估某个技术选型" \
  --worker "researcher:技术面调研" \
  --worker "coder:代码可行性验证" \
  --verifier reviewer --synthesizer writer
# → Swarm root: t_xxx / Workers: t_aaa, t_bbb / Verifier: t_ccc / Synthesizer: t_ddd
```

实测拓扑：root 立刻 `done` 并充当黑板（评论里一条 `[swarm:blackboard] {"key":"topology","value":{...}}`），workers `ready` 并行，verifier `todo` 等全部 workers，synthesizer `todo` 等 verifier。

**这里有个必须提醒的参数坑**：官方文档写的是 `--workers researcher,architect,sre`，但实测 v0.21.3 里这个复数参数不存在，报 `unrecognized arguments: --workers`。正确写法是可重复的 `--worker PROFILE:TITLE[:SKILL]`。照文档抄会直接失败——这也印证了开头那句话：以实测为准。

一条生产形态的流水线通常是四张卡：researcher 采集 → writer 成稿 → reviewer 审校 → publisher 发布。写法上只有一处和 demo 不同——依赖要逐个声明，而不是全挂在同一张父卡下面：

```bash
R=$(hermes kanban create "调研" --assignee researcher --json | jq -r .id)
W=$(hermes kanban create "成稿" --assignee writer --parent $R --json | jq -r .id)
V=$(hermes kanban create "审校" --assignee reviewer --parent $W --json | jq -r .id)
P=$(hermes kanban create "发布" --assignee publisher --parent $V \
      --workspace dir:/abs/blog/content --json | jq -r .id)
```

发布这一跳用 `dir:` 而不是 `scratch`，因为它的产物要留在博客仓库里，不能被完成清理带走。这就是三形态最实际的分工：中间产物用 `scratch`，最终交付用 `dir:`，代码改动用 `worktree`。用错了不会报错，只会在某天发现“文件不见了”——而那时你已经把原因归到模型头上试过三轮了。

### 真实产物怎么落地

```python
# worker 内部（工具调用，不是 shell）
kanban_complete(
    summary="数据采集完成：3 个来源，42 条记录",
    metadata={"sources": 3, "records": 42, "artifact_path": "/abs/path/data.json"},
    artifacts=["/abs/path/data.json"],   # 必须真实存在，否则任务卡在 in-flight
)
```

这里有个容易踩的差异：**路径写在 `metadata` 里不会被上传，必须写在 `artifacts` 参数里**。metadata 是给下游 worker 读的字符串，artifacts 才是持久化的真文件。混用之后的表现是“下游拿不到文件，但元数据里明明有路径”。

顺带一句：summary 会内嵌到 `completed` 事件里（截断 400 字符），notifier 直接用这段文本发通知。所以 summary 的第一句要能独立成立——那是别人唯一会看到的句子。

## 坑与限制

写这一节的目的不是唱衰，是把“文档没写但你一定会踩”的部分标出来。

**未知 assignee 被静默丢弃。** `create --assignee ghost-profile` 会成功，卡停在 `ready`，`dispatch` 打印 `Skipped (non-spawnable assignee — terminal lane, OK)`，`Spawned: 0`。它被当作“控制面 lane”（比如终端型 agent）而不是错误，所以不算 operator-actionable。**更麻烦的是**：文档说会往卡上写一条 `skipped_nonspawnable` 事件，实测连跑两次 dispatch，卡的事件流里只有 `created`，一条都没落库。也就是说，**卡上看不出任何原因**——这恰好是“事件流是唯一事实来源”的一个反例。排查只能靠 `dispatch --dry-run --json | jq .skipped_nonspawnable`、`diagnostics`、`assignees` 三条命令核对拼写。

**gateway 存活检测会假阴性。** 如果 shell 里 `HERMES_PROFILE=researcher` 而 gateway 实际跑在 `default`，`hermes gateway status` 会报“未运行”，`create` 也跟着警告“无 gateway”——但派发其实是正常的。看到警告先看输出里的 `Other profiles` 段。

**stale 回收会误伤长任务。** 4 小时 + 1 小时无心跳才回收，回收是良性的（不计失败、直接回 ready），但当前 run 的进度丢失。任何可能跑超 1 小时的卡，**必须每小时至少调用一次 `kanban_heartbeat`**。

**review 的动词有前置条件。** `request-review` 从 `ready` 也能直接进 review；但 `request-changes` 需要存在活跃 review run，否则报 `task is not in an active review run`——人工打回要用 `reopen-review <id> --reason ...`。另外，当下游已有评审或发布子卡时，父卡应当直接 `complete` 放行，不要再额外 request-review，否则下游被双重门槛卡住。

**`block --kind` 的参数顺序。** `block ID --kind dependency "原因"` 会报 `unrecognized arguments`；必须写成 `block ID "原因" --kind dependency`。另外 `todo` 状态的卡不能被 block，只有 `ready` / `running` 可以。

**`created_cards` 的幻影 id 会被拒。** 完成时声明了不存在、或不是本 worker 创建的卡 id，会被 `HallucinatedCardsError` 拦住：`completion blocked: claimed created_cards that do not exist or were not created by this worker`。这条我只能读源码核实——worker 工具本身不许改别人的卡，实跑不出来。

**显示层小瑕疵。** `show` 只显示 `max-retries`，不显示 `max_runtime_seconds`（虽然 `--max-runtime 90s` 已被接受并生效）。排查超时问题时别靠 `show` 确认。

**边界要说清**：单机单用户是设计选择，不是待办事项。跨机器共享 board 不支持；swarm 在生产中的实际并发规模与成本，没有官方基准，本节不作估计；是否存在 swarm v2 拓扑，官方未提及——这两点均未核实。

## 结语：多 Agent 的下一步不是更复杂的图

回头看开头那个凌晨。真正让你加班的不是“图不够表达”，而是“崩了之后状态在谁手里”。LangGraph 把状态放在你的进程里，所以你必须在每个节点上补幂等；Kanban 把状态放在库里，所以恢复只是“再派一次，让它重读注释”。前者要求你正确，后者允许你不正确。

这也是为什么 `hermes kanban swarm` 只有一张小图、并明确写着“Deliberately no second scheduler”：多 Agent 编排的复杂度不该堆在拓扑上，而该堆在**状态归属、失败分类、人的介入点**这三件事上。图能表达的，迟早会被更强的图表达掉；而一个能在 agent 崩溃、限流、被换人接管之后仍然自洽的队列，才是把 demo 变成生产的那部分。
