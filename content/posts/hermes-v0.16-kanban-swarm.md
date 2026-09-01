---
title: "Hermes Agent v0.16 Kanban Swarm 功能深度解析（面向 AI 开发者）"
date: 2026-09-01
tags: ["hermes-agent","kanban","swarm","多智能体","AI开发"]
author: ChaoZzi
---

## 一、引言：多智能体协作缺的不是模型，是任务编排

假设你要让几个 AI agent 协作完成一篇技术博客：研究者查资料、写作者成文、审查者校对、发布者提交 PR。最省事的做法是让一个 agent 串行干完，或者用 delegate_task 派子任务等结果。规模一上来，问题就暴露了：

- 串行低效：调研阶段就把上下文占满，后续步骤质量下滑；
- 上下文割裂：每个 agent 只看到自己那一段 prompt，交接全靠手工复制粘贴；
- 无任务编排：谁先谁后、失败怎么重试、成果存在哪，全散落在对话里，进程一重启就归零。

Hermes Agent v0.16（2026-06-05 发布）内置的 Kanban Swarm 就是为此设计的：一个 SQLite 持久化的任务板，跨所有 profile 共享，让多个命名 agent 异步协作。每个任务是一行记录，每次交接是一行记录，每个 worker 是带独立身份（profile）的完整 OS 进程。

为什么重要：它把「多智能体协作」从对话里的即兴发挥，变成可查询、可审计、可恢复的持久数据结构。与 delegate_task（RPC 调用：父阻塞等子返回，子代理匿名，失败即失败）相比，Kanban 是持久消息队列加状态机：fire-and-forget，worker 是带持久记忆的命名 profile，可 block、unblock、重跑，崩溃可 reclaim。一句话：delegate_task 是函数调用，Kanban 是工作队列。

## 二、核心概念：卡片、状态机、workspace 与依赖

**任务卡片**是看板的基本单位，一行记录包含标题、正文、assignee（一个真实存在的 profile 名）、状态，以及可选的 tenant、idempotency key。

**状态机**：v0.16 源码（kanban_db.py）定义了 9 个状态：triage（粗略想法的停车场，可自动拆解）→ todo（等父任务完成）→ ready（可被 dispatcher 领取）→ running（worker 运行中）→ blocked（等人工）或 review（待审查）→ done / archived（终态）；另有 scheduled（定时到点才派发）。

为什么重要：状态是 dispatcher 调度和依赖判断的依据，也是人工介入的入口。一个任务卡在哪一步，看板上一目了然。

**workspace 三种类型**决定任务产出放哪：

- scratch（默认）：临时目录，任务完成后删除。交付物必须用 kanban_complete(artifacts=[...]) 声明，才会在清理前复制到持久附件存储；
- dir:<绝对路径>：共享目录（如 Obsidian 库、运营目录），完成后保留，相对路径会被拒绝；
- worktree：git worktree，编码任务专用，完成后保留。

**parents 依赖与阻塞**：task_links 表存 parent→child 边。dispatcher 在所有 parent 都 done 之后，才把子任务从 todo 提升为 ready。父任务的 summary + metadata 会原样注入子任务的上下文（「## Parent task results」），所以父链接既是依赖关系，也是上下文交接通道。

## 三、工作机制：SQLite 看板、dispatcher 与心跳

**SQLite 看板**：任务、运行记录（task_runs）、事件（task_events）全部落库，跨进程、跨重启存活，事后可审计。任务随时可以被不同角色（或人）接手，编排不依赖任何父会话存活。

**dispatcher 调度**：长驻循环，默认每 60 秒一轮（kanban.dispatch_interval_seconds），做三件事：回收 stale 或崩溃的 worker、把满足依赖的 ready 任务提升、原子认领并 spawn 对应 profile 的独立进程（hermes -p <profile>，环境注入 HERMES_KANBAN_TASK / HERMES_KANBAN_BOARD / HERMES_KANBAN_WORKSPACE）。默认内嵌在 gateway 进程里（dispatch_in_gateway: true），独立的 daemon 命令已废弃。注意 assignee 必须是机器上真实存在的 profile，否则 dispatcher 静默不派发，卡片永远躺在 ready 里。

**心跳与超时回收**：长任务应定期调用 kanban_heartbeat，预计超过 1 小时的任务必须每小时至少一次。若任务运行超过 dispatch_stale_timeout_seconds（默认 4 小时）且最近 1 小时无心跳，dispatcher 判定 stale，SIGTERM 掉 worker，任务回 ready 重新派发——不计入失败计数。

为什么重要：心跳把「agent 卡死」和「agent 还在想」区分开，避免一个死循环永久霸占任务；重跑不惩罚，让系统对崩溃有弹性。

## 四、实战上手：命令速查与一个端到端例子

先纠正一个常见误区：hermes kanban 没有 status 子命令，查状态用 list / show / stats。当前版本共 43 个子命令（v0.16 已有 34+，request-review、attach、set-model、repair 等是后来新增的），最常用的几个：

```bash
hermes kanban list                 # 看板总览：每张卡的状态与 assignee
hermes kanban show <task-id>       # 单卡详情：正文、评论、parent 交接、历史 run
hermes kanban stats                # 按状态 / assignee 的统计

# --parent 可重复，声明依赖；--workspace 可选 scratch / worktree / dir:<path>
hermes kanban create "写作：xxx" \
  --assignee writer \
  --parent t_a53f2825 \
  --workspace dir:C:/path/shared \
  --priority 20 \
  --goal                           # goal mode，见第六节
```

create 还有这些常用参数：--triage（进 triage 列，等自动拆解）、--idempotency-key（幂等创建）、--max-runtime / --max-retries、--skill（给 worker 预装技能）、--model / --provider（指定模型）。所有子命令同时可以作为 /kanban <verb> 斜杠命令在网关里使用。

端到端例子就用本文自己这条流水线（本机实测输出）：

```text
$ hermes kanban list
Board: blog (1 other board — `hermes kanban boards list`)
◻ t_c48c72f3  todo      publisher  发布：提交文章到 team/tech-blog（PR→main）
◻ t_944d9d11  todo      reviewer   审查：文章语法与事实核查
◻ t_794dc295  todo      writer     写作：Hermes Agent v0.16 Kanban Swarm 功能深度解析（约2000字）
● t_a53f2825  running   researcher 调研：Hermes Agent v0.16 Kanban Swarm 功能（面向 AI 开发者）
✓ t_d81b5fc4  done      orchestrator Hermes Agent v0.16 Kanban Swarm 功能深度解析（面向 AI 开发者）
```

orchestrator 卡片 done 后，researcher → writer → reviewer → publisher 依次解锁。你正在读的这篇正文，就是 writer 环节在这个共享 workspace 里写出来的。如果只想快速建一个「并行调研 + 汇总」拓扑，hermes kanban swarm 一条命令搞定：--worker 定义多张并行卡，--verifier / --synthesizer 定义汇总结点，共享上下文以结构化 JSON 评论存在 root 卡上。

## 五、协作与 review 流程：profile 分工与 block 语义

**profile 分工**：每个角色是一个独立 profile，各有自己的记忆与技能。本文流水线用了 orchestrator / researcher / writer / reviewer / publisher 五个。worker 完成时通过 kanban_complete(summary, metadata) 留下结构化交接，下游自动收到。

**review 流程**：实现完成后调用 kanban_request_review(summary, metadata, reviewer?) 把任务移入 review 列（注意：不是 block）。reviewer 认可则 kanban_complete 收尾，打回则 kanban_request_changes(reason) 归还实现者。默认 reviewer 带 sdlc-review 技能。需要留意：v0.16 已有 review 状态，但 request-review / request-changes 命令是后续版本新增的。

**block 的四种语义**（--kind 参数）：

- dependency：回 todo 等父任务，自动恢复，无需人工；
- needs_input：需要人类回答；
- capability：硬墙（缺权限、缺凭据），agent 自己解决不了；
- transient：暂时性故障，稍后重试。

为什么重要：block 不是简单的「暂停」，它决定任务回流到哪里。同因 unblock→reblock 累计 2 次会被自动路由进 triage 防循环（计数器只有 complete 才重置），所以别拿 block 当重复 review 用。

## 六、高级特性：goal mode、hotspot 与 artifacts

- **goal mode**（--goal）：worker 在同一会话内循环，每轮由辅助 judge 对照卡片 title + body 判断是否完成，直到完成或预算耗尽（默认 20 轮，--goal-max-turns 调整）；预算耗尽则 block 待人工，而不是静默退出。适合开放式、多步任务。
- **hotspot 冲突检测**：约定而非强制。worker 发现与兄弟任务反复冲突同一文件时，在卡片发「hotspot: <path> — 原因」前缀评论并写入完成 metadata；orchestrator 看到两张以上同名 hotspot 评论，应建「拆文件」重构卡。
- **artifacts 交付**：kanban_complete(artifacts=[绝对路径]) 在 scratch 清理前把交付物复制到持久附件存储，缺失则任务保持 in-flight。kanban_attach（base64）/ kanban_attach_url（URL 抓取）上限 25MB；complete 时声明的 created_cards 会被内核校验存在性，幻影 id 直接拒绝完成。
- **集成**：cron 脚本可用 --idempotency-key 幂等创建任务（同 key 返回既有卡）；notify-subscribe 把终态事件（completed / blocked / gave_up / crashed / timed_out）推送到网关聊天。

## 七、适用场景、最佳实践与总结

Kanban Swarm 适合五类场景：研究 triage（并行研究者 → 分析师 → writer）、定时运营（cron + 同 profile + 共享 dir）、数字孪生（常驻命名助手积累记忆）、工程流水线（拆解 → 并行 worktree → review → PR），以及 fleet 管理（一个 profile 管 N 个对象）。

但边界要清楚：这是单机设计，本地 SQLite + 本机 PID 检测，不支持跨主机共享板（每台主机独立 board，跨机用 delegate_task 或消息队列桥接）；调度粒度是 60 秒 tick；scratch 完成即删；dashboard 插件默认无鉴权，别用 --host 0.0.0.0 暴露到公网。

最佳实践：设计决策（命名、schema、文件格式）在拆解前由 orchestrator 定死，并写进每张子卡；用 parents 表达依赖，而不是写在正文里；长任务记得心跳；交付物一律走 artifacts；同文件冲突尽早发 hotspot。

回到开头的问题：delegate_task 解决「派个活」，Kanban Swarm 解决「组织一群人干活」。当你的多智能体应用开始需要流程、交接和审计时，它补上的正是这块地基。
