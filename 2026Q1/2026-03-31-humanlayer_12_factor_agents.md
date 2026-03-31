# 12 Factor Agents：面向生产环境的 Agent 工程原则

- **来源:** https://www.humanlayer.dev/blog/12-factor-agents
- **作者:** Dex
- **日期:** 2025-04-03
- **出处:** HumanLayer Blog
---

## 概览
这篇文章可以看成是把“12 Factor App”的思路迁移到 agent 时代：**agent 不是一团神秘的 autonomous magic，而是可以被工程化、被拆解、被运营的软件系统。**

作者对很多流行 agent 框架其实挺不客气，核心观点是：真正能放到生产环境、直接面对客户的 agent，往往不是“给一个 prompt + 一堆 tools + loop 到完成”为主，而是**大量确定性软件 + 少量关键 LLM 决策点**的组合。

一句话说，这篇文章是在劝你：**别迷信 agent framework，先把 agent 当成软件系统来设计。**

## 文章主线

### 1. Agent 不是要替代软件，而是把决策点语言化
作者先回顾了软件演化：
- 传统程序本质上是 directed graph
- 后来有 Airflow / Prefect 这类 DAG orchestrator
- 再后来是在 DAG 里塞一些 ML/LLM 节点

Agent 的“承诺”是：你不再手写每一条边和每个分支，而是把一部分决策边界交给 LLM 来实时决定。但这并不意味着别的工程基础可以丢掉。

### 2. 真正有效的不是大一统万能 agent，而是 micro agents
作者明显更偏向 **small, focused agents**，而不是一个上下文无限膨胀、什么都想做的通用大 agent。

原因很朴素：
- 上下文越长越容易失焦
- 工具越多越容易选错
- 任务越宽泛越难引入人类反馈
- 长 loop 更容易掉进 context error loops

所以作者主张把 agent 设计成一组职责清晰、边界明确的小 agent，而不是一个巨型 all-in-one loop monster。

## 12 个 factor 的核心意思

## Factor 1: Natural Language to Tool Calls
LLM 最核心的价值之一，是把自然语言意图翻译成结构化动作。也就是：
- 人说目标
- 模型决定下一步调用什么工具 / 做什么动作
- 系统执行并返回结果

这个 factor 本质上是在说：**LLM 最适合做的是“决策接口”，不是包办整个系统。**

## Factor 2: Own Your Prompts
Prompt 不应该完全外包给框架。你得自己掌控：
- system prompt 写了什么
- 工具描述怎么写
- 状态怎么注入
- prompt 版本如何演化

作者的潜台词是：prompt 本身就是产品逻辑的一部分。把它藏在框架黑盒里，后面调优会很痛。

## Factor 3: Own Your Context Window
这个 factor 很关键。agent 的每一步输入，本质上都是：
> 到目前为止发生了什么，下一步该做什么？

所以 context window 不只是聊天记录，而是 agent 的工作记忆。你必须自己控制：
- 放哪些状态
- 哪些错误要保留
- 哪些 observation 要压缩
- 什么时候截断、什么时候摘要

这和 Manus 那篇 context engineering 的思路是同一个方向：**上下文不是附属品，它就是 agent 的运行时。**

## Factor 4: Tools Are Just Structured Outputs
这个观点我很认同：所谓 tools，本质上就是模型输出的一种结构化形式。

也就是说，tool calling 不是神秘机制，而是：
- 模型输出符合 schema 的 action
- 系统拿这个 action 去执行
- 再把 observation 回灌给模型

这会带来两个工程好处：
- 你会更重视 schema / 参数设计
- 你会把 tools 看成稳定接口，而不是 prompt 魔法

## Factor 5: Unify Execution State and Business State
Agent 有两种状态：
- **execution state**：当前走到哪一步、暂停在哪、上次 tool call 是什么
- **business state**：用户任务本身的进展、产物、领域对象

作者认为这两者不要割裂太厉害，否则恢复、调试、重跑都会变复杂。更好的方式是设计一套统一、可持久化、可恢复的状态模型。

这点很工程，也很重要：**agent 如果不能可靠恢复，就很难进生产。**

## Factor 6: Launch/Pause/Resume with Simple APIs
生产环境里的 agent 不能只会“一口气跑完”，还要能：
- 启动
- 暂停
- 恢复
- 查询状态

这意味着 agent 不该只是一个 blocking function call，而应该是带生命周期的任务系统。尤其一旦接入人类审批、异步工具、长时任务，这个能力就是基础设施。

## Factor 7: Contact Humans with Tool Calls
联系人工不应该是架构外特判，而应该也是系统里的一个标准动作。

也就是把“找人确认 / 询问补充信息 / 请求审批”当作一种 tool call。这样设计的好处是：
- 人类在环成为 agent loop 的自然组成部分
- 审批 / 追问 / 升级都有一致接口
- 更容易把 agent 放进真实业务流程

这个 factor 本质在说：**human-in-the-loop 不是失败兜底，而是原生能力。**

## Factor 8: Own Your Control Flow
作者明确反对把控制流全丢给框架或模型。你应该自己掌控：
- 什么时候 loop
- 什么时候强制结束
- 什么时候切人工
- 什么时候进入特定分支

这不是在否定 agent，而是在强调：**控制流属于系统设计，不该完全外包给模型。**

## Factor 9: Compact Errors into Context Window
错误不能简单吞掉。应该把错误压缩成对模型有用的形式放回上下文里，让它能基于失败继续调整。

这和 Manus 那篇“Keep the Wrong Stuff In”几乎是同一路数：
- 失败是信息
- 错误是证据
- agent 的价值之一就是恢复能力

区别在于这篇更强调“compact”——别把一整坨冗长报错原样塞进去，而是压成能指导下一步决策的上下文。

## Factor 10: Small, Focused Agents
这是全文最强烈的主张之一。作者认为小而专注的 agent 几乎总是更实用，因为：
- context 更短
- 目标更清晰
- 调试更容易
- 工具集合更小
- 更容易插入实时人工反馈

对抗“大 agent 幻觉”的办法，不一定是更强模型，很多时候是**更窄职责边界**。

## Factor 11: Trigger from Anywhere
生产环境里的 agent 不该只能从一个聊天入口触发。它应该能从各种事件源被调用，比如：
- 用户消息
- webhook
- cron
- 后台状态变化
- 外部系统事件

也就是说，agent 应该像普通服务一样被系统集成，而不是只存在于某个聊天框里。

## Factor 12: Make Your Agent a Stateless Reducer
这个 factor 很像现代后端系统设计理念：让 agent 尽量像一个 stateless reducer。

也就是：
- 输入当前状态 + 新事件
- 输出下一步 action / 新状态
- 真正的状态持久化在外部系统里

这样做的好处是可测试、可恢复、可重放、可审计。换句话说：**把 LLM 尽量限制在纯决策层，而不是让它背着全部运行状态。**

## 我觉得这篇最有价值的地方

### 1. 把 agent 从“神秘智能体”拉回“可维护软件”
这篇文章最好的地方就是去魅。它不鼓吹 autonomous magic，而是强调：
- prompt 要自己管
- context 要自己管
- control flow 要自己管
- state 要自己管

### 2. 小 agent 思路非常实战
现实里很多 agent 做坏了，不是模型不够强，而是职责边界太宽、上下文太脏、工具太多、循环太长。

### 3. human-in-the-loop 被放到了架构中心
不是最后补个“人工审核”按钮，而是把联系人工本身设计成系统里的正规动作。这点对业务落地特别重要。

### 4. 错误恢复被当成一等公民
这比只看 task success 更像真实世界。生产环境里真正好用的 agent，不是从不犯错，而是**犯错后能被系统安全地拉回正轨。**

## 和其他文章的呼应
这篇和你刚放进去的几篇其实能串起来看：
- 和 Manus 那篇一致：都强调 **context / errors / agent runtime** 是工程核心
- 和 Anthropic 那篇 SRE agent 一致：都强调 **tools、control、human-in-the-loop、受控执行边界**
- 和 Tw93 那篇 agent 工程实践也一致：**agent 成败主要看系统设计，不是只看模型参数**

## 一句话总结
这篇文章的核心就是：**真正能上线的 agent，不是一个会无限循环的 prompt，而是一套由 prompts、context、tools、state、control flow、human feedback 和 lifecycle APIs 共同组成的软件系统。**

如果只记一句，我会记这个：**agents are software first，LLMs 只是其中负责决策的那一层。**
