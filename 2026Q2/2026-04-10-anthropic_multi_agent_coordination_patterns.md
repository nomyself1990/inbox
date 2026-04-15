# Anthropic：多 Agent 协调模式，五种做法与适用场景

- **来源:** https://claude.com/blog/multi-agent-coordination-patterns
- **作者:** Cara Phillips（并有 Eugene Yan、Jiri De Jonghe、Samuel Weller、Erik S. 贡献）
- **日期:** 2026-04-10
- **出处:** Anthropic Blog
---

## 概览
这篇文章是 Anthropic 对 **multi-agent system coordination patterns** 的一个很实用的框架化总结。重点不在“多 agent 很酷”，而在：**如果你已经决定要用多 agent，那么该用哪种协作结构，什么时候该升级，什么时候其实不该升级。**

如果只记一句话：**别先追求复杂架构，先用最简单能工作的模式，再根据真实瓶颈往上演化。默认起点通常是 orchestrator-subagent。**

## 文章想解决什么问题
Anthropic 先点了一个很常见的误区：很多团队选 coordination pattern，不是因为问题真的需要，而是因为某种结构“听起来更高级”。

他们的建议非常工程化：

- 从最简单可行的模式开始
- 观察它具体卡在哪里
- 再决定要不要演化成更复杂的模式

这篇文章整理了五种典型模式：

1. **Generator-verifier**
2. **Orchestrator-subagent**
3. **Agent teams**
4. **Message bus**
5. **Shared state**

## 1. Generator-verifier
这是最简单、也最常见的多 agent 模式之一。

### 怎么工作
- 一个 **generator** 先产出结果
- 一个 **verifier** 负责检查是否符合标准
- 不合格就把反馈打回去
- generator 根据反馈重做
- 循环直到通过，或者达到最大迭代次数

### 适合什么场景
适用于：**输出质量要求高，而且评价标准能说清楚**。

文章给的典型例子：

- 客服回复生成 + 准确性/语气/完整性校验
- 代码生成 + 测试验证
- fact-checking
- rubric-based grading
- 合规性检查

本质上适合这类任务：**错一次代价高于多跑一次生成循环。**

### 会卡在哪里
关键问题不是“有没有 verifier”，而是：**verifier 的标准是否足够明确。**

如果 verifier 只是被要求“看看好不好”，那它大概率只是在走形式，无法真正拦住坏结果。

另一个问题是：

- 如果“评估”本身和“生成”一样难
- verifier 其实也未必真能看出问题

还有一个工程坑：generator 和 verifier 可能会进入来回打架、迟迟不收敛的循环，所以必须加：

- 最大迭代次数
- fallback 策略
- 人工升级路径 / 带 caveat 的最佳尝试返回

## 2. Orchestrator-subagent
这篇文章里，Anthropic 明确把它当作 **默认起点**。

### 怎么工作
- 一个主 agent / orchestrator 接收总任务
- 它决定如何拆解
- 某些部分自己做
- 某些部分交给 subagent
- subagent 返回结果后，再由 orchestrator 汇总成最终输出

文章直接说 **Claude Code** 就是这个模式：主 agent 保持主线推进，遇到独立问题时把探索型工作丢给 subagent，在后台并行完成，再把蒸馏结果收回来。

### 适合什么场景
适用于：**任务拆解清晰，子任务边界明确，而且相互依赖不强。**

例子：自动 code review 系统，把安全检查、测试覆盖率、代码风格、架构一致性分别交给不同 subagent。

为什么这个模式适合当默认起点？因为它兼顾了：

- 总体目标的一致性
- 局部探索的并行化
- 上下文隔离
- 实现复杂度相对可控

### 会卡在哪里
最核心的问题是：**orchestrator 会变成信息瓶颈。**

如果某个 subagent 发现了另一个 subagent 也该知道的信息，这条信息必须绕回 orchestrator，再由 orchestrator 判断和转发。随着 handoff 增加，关键细节很容易丢。

另一个问题是：

- 如果实际上没有显式并行
- subagent 只是一个接一个执行
- 那你付了多 agent 的成本，却没拿到速度收益

## 3. Agent teams
这个模式和 orchestrator-subagent 看起来有点像，但核心区别是：**worker 是持续存在的，不是一次性调用。**

### 怎么工作
- 一个 coordinator 启动多个 teammate agents
- 这些 agent 作为独立进程长期存在
- 从共享任务队列领任务
- 持续处理多步工作
- 在多个 assignment 中保留自己的上下文和领域熟悉度

### 适合什么场景
适用于：**并行、相互独立、而且需要持续多步推进的长任务。**

文章的例子是大型代码库框架迁移：

- 每个服务由一个 teammate 独立迁移
- 各自处理依赖、代码修改、测试修复、验证
- coordinator 最后做系统级集成验证

这里的价值在于：worker 不会每次都“重新认识世界”，而是能逐渐积累对自己那一块领域的理解。

### 会卡在哪里
这个模式最严格的前提是：**任务必须足够独立。**

如果一个 teammate 的工作会影响另一个 teammate，而他们彼此又无法自然同步中间发现，就会：

- 输出冲突
- 互相踩改动
- 局部最优但全局不一致

尤其当多个 teammate 操作同一套代码、数据库或文件系统时，冲突会更明显。所以这个模式依赖：

- 精细任务切分
- 资源隔离
- 冲突解决机制

## 4. Message bus
这是一个更偏事件驱动系统的模式。

### 怎么工作
- agent 不再彼此直接协调
- 大家通过一个 message bus 进行发布 / 订阅
- router 按 topic 或语义把消息分发给相应 agent
- 新 agent 可以按能力接入，不需要重写既有连接关系

### 适合什么场景
适用于：**event-driven pipeline，而且 agent 生态会持续扩张。**

Anthropic 给的例子是安全运营自动化：

- 各种告警进来
- triage agent 先分类
- 不同类型告警分派给不同调查 agent
- 调查过程中再触发 enrichment / response 等后续动作

这种情况下，流程不是完全预先写死的，而是随着事件类型、发现结果和新能力接入动态生长。

### 会卡在哪里
这个模式的痛点主要是可观测性和调试性：

- 一个事件触发多轮级联后，很难追踪到底发生了什么
- 如果 router 路由错了、漏了，系统可能“静默失败”
- 用 LLM 做语义路由虽然灵活，但也引入新的不确定性

所以 message bus 很强，但不是白送的。它往往需要更完整的：

- logging
- tracing
- correlation
- routing audit

## 5. Shared state
这是最去中心化的一种模式。

### 怎么工作
- 没有中央 orchestrator，也没有 message router
- 多个 agent 直接围绕一个共享持久化存储协作
- 这个存储可以是数据库、文件系统或文档
- agent 自主读取、写入、基于别人留下的结果继续工作

### 适合什么场景
适用于：**协作型任务，agent 之间需要实时共享发现并在彼此成果上继续推进。**

文章例子是 research synthesis：

- 一个 agent 查学术论文
- 一个 agent 看行业报告
- 一个 agent 看专利
- 一个 agent 盯新闻
- 某个 agent 的发现会立即影响其他 agent 的搜索方向

这时 shared state 很自然，因为知识是在不断累积和相互增强的。

### 会卡在哪里
最大的挑战不是工程层面的锁和并发，而是**行为层面的失控**。

常见问题：

- 重复劳动
- 互相追逐同一条线索
- 结论冲突
- reactive loop：A 写了东西，B 响应，A 再响应 B，系统一直烧 token 但没收敛

文章这里点得很对：

- 并发写入、版本控制、分区这些是工程问题
- **reactive loop 是行为问题**

所以 shared state 必须从设计一开始就定义 termination condition，比如：

- 时间预算
- 收敛阈值
- 连续 N 轮没有新发现
- 指定一个 agent 判断“答案已经够了”

## 文章给的选择逻辑
Anthropic 没把这五种模式讲成一张静态表，而是重点讲了几组“怎么分辨我该选哪边”。这部分很值得记。

### Orchestrator-subagent vs Agent teams
核心问题：**worker 是否需要跨多轮保留上下文。**

- 子任务短、边界清晰、一次出结果：选 **orchestrator-subagent**
- 子任务长、多步、持续积累领域上下文：选 **agent teams**

### Orchestrator-subagent vs Message bus
核心问题：**流程结构是否可预测。**

- 步骤顺序基本固定：选 **orchestrator-subagent**
- 路径由事件动态决定、未来还会长出新分支：选 **message bus**

### Agent teams vs Shared state
核心问题：**worker 之间是否需要共享中间发现。**

- 各自干各自的，最后合并：选 **agent teams**
- 要实时借用彼此发现继续推进：选 **shared state**

### Message bus vs Shared state
核心问题：**系统围绕“事件流转”还是“知识积累”组织。**

- 阶段式事件驱动：选 **message bus**
- 共同维护一个不断增长的知识库：选 **shared state**

## Anthropic 的结论
文章最后的建议非常明确：

- 生产系统里，常常会混合使用多种模式
- 这些模式不是互斥选项，而是 building blocks
- **对大多数 use case，建议先从 orchestrator-subagent 开始**

原因也很合理：它覆盖面最广，协调成本最低，足够多的问题都能先跑起来。等真实瓶颈出现，再往：

- generator-verifier
- agent teams
- message bus
- shared state

这些方向演化。

## 我觉得最值得记的点

### 1. 先从结构问题出发，而不是从“agent 个数”出发
这篇文章最好的一点，是它把“多 agent 架构设计”从玄学拉回了系统结构问题：

- 任务能不能清晰拆分
- worker 要不要长期保留上下文
- 中间发现要不要互通
- 流程是不是事件驱动
- 能不能容忍中心节点

### 2. 默认起点不是最先进的，而是最稳的
Anthropic 并没有鼓吹最花哨的模式，而是明确说：**多数情况下先用 orchestrator-subagent。** 这很务实。

### 3. 共享状态和消息总线都很强，但都比看上去更难
很多人会把 shared state / message bus 想成“更高级的多 agent”。文章提醒得很对：它们带来的不是免费能力，而是更高的调试、收敛和治理成本。

### 4. 终止条件是 shared-state 系统的一等公民
这是一个特别容易被忽视、但非常关键的点。没有 termination design 的 shared-state，多半会变成无限燃烧 token 的系统。

### 5. 多 agent 的真正难点是信息流，而不是 agent 本身
从这五种模式看，本质差别其实都在：

- 信息怎么流动
- 上下文怎么切边界
- 谁来决定下一步
- 什么叫完成

## 一句话总结
**Anthropic 这篇文章把多 agent 协作讲得很落地：别先迷信复杂模式，先用 orchestrator-subagent 起步；只有当你真的遇到质量校验、长期并行、事件驱动或共享发现这些结构性需求时，再分别演化到 generator-verifier、agent teams、message bus 或 shared state。**
