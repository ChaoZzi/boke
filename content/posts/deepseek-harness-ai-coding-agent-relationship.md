---
title: "从 DeepSeek Harness 聊起：AI 编程工具、Agent 与 Harness 的关系厘清"
date: 2026-09-02
tags: ["deepseek-harness","hermes-agent","claude-code","codex","qoder","AI编程","agent"]
author: ChaoZzi
---

> 本文整理自一次围绕 DeepSeek Harness 的问答对话：它和 Hermes Agent 是什么关系？它算不算 Qoder / Claude Code / Codex 那类编程工具？AI 编程工具与 Agent 的概念关系是什么？四个问题，一条主线：**Agent 是引擎，编程工具是装好引擎的车，Harness 是引擎舱。**

## 一、DeepSeek Harness 和 Hermes Agent 有什么关系？

**核心结论：两者没有任何组织或代码上的关系——是同一赛道（Agent Harness / 智能体运行时）的竞品，且在实际使用中天然互补：Hermes Agent 可以作为 DeepSeek 模型的前端（运行 Hermes 的会话完全可以用 DeepSeek 模型当大脑）。**

| 维度 | DeepSeek Harness (dsh) | Hermes Agent |
|---|---|---|
| 出品方 | DeepSeek AI（深度求索） | Nous Research |
| 本质 | 开源 Agent Harness，编码智能体框架 | 通用 Agent 运行时 / 桌面平台 |
| 架构哲学 | "Everything is a plugin"，基于 Cordis 元框架，模型/工具/技能/会话/沙箱全可替换 | 技能(Skills)+插件+工具集+MCP 的可组合架构，多平台网关 |
| 发布状态 | v0.1 developer preview（2026-08 随 V4-Pro 发布），MIT，明示"会有破坏性变更" | 成熟产品 |
| 竞品参照 | 官方对标 Claude Code / OpenAI Codex | 同为该阵营一员 |

关系要点：

1. **同类竞品**：DeepSeek 官方框架 "Agent = Model + Harness"，发布 dsh 是想从"只卖模型"上移到"掌控模型周围的工具/循环/编排层"——这与 Nous 做 Hermes 的动机同构，两者都在抢占同一个生态位。
2. **无血缘**：深度求索和 Nous Research 是两家独立机构，dsh 不包含任何 Hermes 代码，Hermes 也不依赖 dsh。
3. **可互操作（单向已实锤）**：Hermes 是模型无关的，Hermes + DeepSeek 模型（provider: deepseek）是常见组合。反过来，dsh 也支持配置 Anthropic/OpenAI/自定义兼容端点，理论上可接 Nous 的模型。
4. **生态层可叠加**：Hermes 可以把 Claude Code / Codex 等编码 CLI 当 subagent 委托（有专门 skills），dsh 同样具备 delegate 到 Claude Code/Codex 的能力。两边把对方当"工具"调用在架构上都不受阻。

## 二、DeepSeek Harness 可以理解为像 Qoder / Claude Code / Codex 这样的编程工具吗？

**方向对，但差半级：可以理解为"编程工具"，不过它是 Claude Code / Codex 那一类里更"底层"的一种——它们是成品车，DeepSeek Harness 是"整车可拆装图纸 + 可换零件"的成品车。**

| 工具 | 类别 | 形态 | 绑定模型 |
|---|---|---|---|
| Qoder（阿里） | Agentic 编程工具 | VSCode 改造的 IDE/工作台 | 通义系为主 |
| Claude Code（Anthropic） | Agentic 编程工具 | 终端 CLI + IDE/桌面 | Claude |
| Codex（OpenAI） | Agentic 编程工具 | CLI + IDE + 云 | OpenAI |
| DeepSeek Harness (dsh) | Agentic 编程工具 + 可组装框架 | 本地 Web UI / headless / Python SDK | 模型无关（DeepSeek/Anthropic/OpenAI/自定义端点均可） |

为什么说"对但不完全对"：

1. **日常使用体验确实同类**：dsh 就是 DeepSeek 版的编码 agent——能看仓库、改文件、跑 shell、搜文件/网页、维护计划、调用 skills、派 subagent、审批敏感操作。官方也明确拿 Claude Code / Codex 当参照竞品。
2. **但 "harness" 一词是刻意的**：DeepSeek 的定义是 *Agent = Model + Harness*（模型 + 挽具）。他们卖的不是"一个固定的编程助手"，而是"包住模型的那一整层系统"，且这层系统里每个零件（模型、工具、技能、会话、沙箱、编排、UI）都能换。
3. **它能"骑"在其他编程工具上面**：dsh 支持把 Claude Code、Codex 当 subagent 调进自己的编排流程——与其说是那些工具的竞品，不如说是能坐在它们之上的框架。
4. **成熟度落差明显**：Qoder/Claude Code/Codex 是稳定商业产品；dsh 是 v0.1 developer preview，仓库自己写着 "THERE WILL BE COMPATIBILITY-BREAKING CHANGES"。

一句话记忆点：Qoder ≈ IDE 型成品；Claude Code/Codex ≈ 终端型成品；dsh ≈ 想成为"一切 Agent 的底座"的开源半成品——当下当编程工具用没问题，长期它的野心是让你在上面盖自己的工具。

## 三、AI 编程工具和 Agent 有什么关系？

**核心结论：AI 编程工具 ≈ 智能体（Agent）在"写代码"这一垂直场景的成品化落地——Agent 是引擎，编程工具是装好引擎的整车。不是并列关系，是"通用技术 → 行业产品"的实例化关系。**

一句话模型，Agent 是个循环：

```
Agent = 模型(大脑) + Harness(循环骨架) + 工具(手脚)
         │              │                    │
     LLM 决策       规划→调用→观察→再规划     文件系统、shell、
                                          git、浏览器、执行器
```

Agent 的本质是**能自主多步执行**的系统：给它一个目标，它自己拆步骤、调工具、看结果、错了重来，直到完成。Claude Code、Codex、dsh、Qoder 的核心，就是这个循环。

三层嵌套关系：

1. **Agent（通用概念）**——任何"有目标、会调工具、能迭代"的 AI 系统。Hermes Agent 就是通用型：写信、查资料、跑代码什么都能干。
2. **Coding Agent（专用智能体）**——把上面的循环绑上一套编程专用工具集：读/改文件、跑 shell、grep 搜索、git 操作、跑测试。模型被训练成"以代码操作为母语"。
3. **AI 编程工具（产品形态）**——把 coding agent 包上界面和分发：Claude Code 的终端 CLI、Qoder 的 IDE、dsh 的本地 Web UI + 审批流 + 沙箱。

所以：**所有主流 AI 编程工具本质上都是一个 coding agent 的"产品化包装"**，它们之间的差别主要在包装层（界面、模型绑定、生态、成熟度），循环骨架大同小异。

两个反例让边界更清楚：

- **Agent ≠ 编程工具**：Hermes、客服 agent、浏览器 agent、数据分析 agent 都是 Agent，但不写代码。
- **编程工具 ≠ Agent**：早期 Copilot 补全、代码搜索、"选中→解释"这类是辅助（assistive），没有自主多步执行，不算 agent——所以行业现在特别强调 "agentic coding" 这个词，指的就是后者。

为什么 DeepSeek 管自己的东西叫 "Harness"（挽具）？这个命名恰恰暴露了关系：模型本身不会干活，是"挽具"（循环、工具、权限、记忆那层软件）套上去，它才变成能拉车的 Agent。DeepSeek 认为模型会持续更替、挽具才是护城河，所以把挽具整个开源、全部做成插件——**编程工具之争，争的就是"谁的挽具套得更舒服、更可换"**。Hermes 也是同一层的东西，只是没把赌注押在编程单场景上。

```
        ┌─────────────────────────────────────────┐
        │  Agent（通用智能体）· 概念层              │
        │   ┌──────────────────────────────────┐  │
        │   │  Coding Agent · 能力层            │  │
        │   │  ┌────────────────────────────┐  │  │
        │   │  │ 产品层：CLI/IDE/Web/审批流  │  │  │
        │   │  │ Claude Code·Codex·Qoder·dsh│  │  │
        │   │  └────────────────────────────┘  │  │
        │   └──────────────────────────────────┘  │
        │  (Hermes Agent 也在这层，但不只编程)      │
        └─────────────────────────────────────────┘
```

## 四、编程工具其实就是"专门写代码的 agent + 界面包装"吗？

**基本正确，但"界面包装"说小了——准确讲是：Coding Agent + 三层外壳（工具集、权限与审批、界面与生态）。界面只是最不值钱的那层。**

更精确的拆解公式：

```
AI 编程工具 = 模型(大脑)
            + Agent 循环(规划→工具→观察→迭代)
            + 编程专用工具集(文件读写/shell/git/搜索/跑测试)   ← 使它"会写代码"
            + 权限与沙箱(审批敏感操作、隔离环境)               ← 使它"敢干活"
            + 界面与生态(CLI/IDE/Web、账户、插件市场、记忆)    ← 使它"好用"
```

容易漏掉的是中间两层——**编程工具集**和**审批沙箱**，它们才是"编程"之所以是"编程"、以及商业上最重的部分：

1. **工具集决定能力边界**：普通 agent 只有浏览器/搜索这类通用工具；编程工具换成了精确的代码工具——行级编辑、语义搜索、编译报错回读、测试执行。模型再聪明，工具不准也写不了真项目。
2. **沙箱与审批决定可用性**：让 AI 跑 `rm -rf` 还是只准改指定文件？这层管住"敢不敢把 agent 放进真实仓库"。Claude Code 的权限系统、dsh 可配置的沙箱插件，都是产品级的重投入。
3. **界面反而是最容易抄的**：ChatGPT 的 Web 界面人人都能仿，但没人在乎——壁垒在 agent 循环的质量、上下文管理（读十万文件不迷失）、工具链的打磨。

用 dsh 的视角校准一下："everything is a plugin"——意思是连上面公式里的**每一层都可以换**：模型可换、工具可换、循环可换、UI 可换。它的存在本身就证明：编程工具 = 一堆可插拔零件按"coding"场景的默认装配，**没有哪一层是神圣不可拆的**。

修正版总结：编程工具 = 会写代码的 Agent（模型+循环+代码工具集+沙箱）+ 产品外壳（界面、审批流、账户生态）。简化成"agent + 界面"没错，只是界面在括号里只占最后一个词的分量。

## 五、对 AI Agent 开发者的选型启示

概念理清之后，选型逻辑自然浮现：

- **要卖"工具"** → 在 Coding Agent 上加垂直包装（Qoder 路线）；
- **要卖"能力"** → 做通用 Agent + 编程作为其中一个技能（Hermes 路线）；
- **要自研** → 别从零写循环，站在 dsh（可插拔骨架）或成熟 SDK 上，只写你自己的"挽具零件"；
- **dsh 值得试但不值得押注**：目前唯一模型无关的同类开源方案，可接已有 API key 跑；但处于 preview 期、破坏性变更频繁，别作为核心工作流依赖。

## 参考来源

- [github.com/deepseek-ai/deepseek-harness](https://github.com/deepseek-ai/deepseek-harness) — dsh 官方仓库：MIT、Cordis 插件架构、developer preview
- [deepseek.com/harness](https://www.deepseek.com/harness/en/) — 官方开发者预览页（"Everything is a plugin. Every run is traceable."）
- [VentureBeat: DeepSeek Harness launches as open source rival to Claude Code](https://venturebeat.com/technology/deepseek-harness-launches-as-open-source-rival-to-claude-code-alongside-v4-pro-on-api-with-higher-prices) — dsh 与 Claude Code/Codex 逐项对比
- [MindStudio: What Is DeepSeek Harness?](https://www.mindstudio.ai/blog/deepseek-harness-agentic-coding) — "能坐在 Claude Code/Codex 之上的框架"的评测
- [qoder.com](https://qoder.com/zh) — 阿里 Qoder 官方
- [hermes-agent.nousresearch.com/docs](https://hermes-agent.nousresearch.com/docs) — Hermes Agent 官方文档
