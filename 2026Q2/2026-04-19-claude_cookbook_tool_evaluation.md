# Claude Cookbook：Tool evaluation，用评测驱动工具设计与优化

- **来源:** https://platform.claude.com/cookbook/tool-evaluation-tool-evaluation
- **作者:** Anthropic
- **日期:** 2025-09-10
- **出处:** Claude Cookbook
---

## 概览
这篇 cookbook 不是泛泛讲 eval，而是给了一个非常具体的模式：
**把“工具好不好用”单独拎出来评测，而不是把它混在整个 agent 系统效果里靠感觉判断。**

核心做法是：

- 准备一批 evaluation tasks
- 让多个 agent 独立跑这些任务
- 记录它们的 tool calls、耗时、准确率、summary、feedback
- 用这些结果反推工具定义、参数、描述、错误信息、返回格式哪里该改

一句话说：**这是一个专门评工具 ergonomics 和 tool-use effectiveness 的 eval harness。**

## 这篇在做什么
页面给的是一个完整的 Python 示例，目标很明确：

- 从 XML evaluation file 里读取任务
- 给 agent 一个固定的 evaluator prompt
- 用简单 agent loop 跑任务
- 当模型触发 `tool_use` 时执行对应工具
- 记录 tool metrics
- 最后生成 report

这个模式有两个重点：

### 1. task 和 tools 解耦
任务来自独立的 evaluation file，不是硬编码在 agent 逻辑里。
这样你可以：

- 不改 harness，只换任务集
- 不改任务，只换工具定义
- 在同一批任务上比较不同工具版本

### 2. 不只评结果，还评过程
除了最终 `response == ground truth` 之外，它还要求 agent 输出：

- `<summary>`：做了哪些步骤、为什么这么做、各工具输入输出是什么
- `<feedback>`：工具名字、参数、描述、报错、token 消耗、改进建议
- `<response>`：最终答案

也就是说，这个 harness 同时在产出：

- **定量结果**：accuracy、duration、tool calls
- **定性反馈**：agent 对工具可用性的主观“吐槽”

## Prompt 设计的重点
示例里的 `EVALUATION_PROMPT` 很值得看，因为它把 agent 限定成了一个“会做事也会复盘的 evaluator”。

它明确要求 agent：

1. 必须用工具完成任务
2. 用 `<summary>` 包步骤总结
3. 用 `<feedback>` 包工具反馈
4. 用 `<response>` 包最终结果

而且对 summary / feedback 都有明确要求，比如：

- 工具按什么顺序调用
- 为什么调用
- 输入是什么
- 输出是什么
- 工具名是否清晰
- 参数是否清楚
- description 是否准确
- 是否报错 / 是否返回太多 token
- 如何改会更好

这背后的意义很实际：
**你不只是想知道 agent 做对没做对，还想知道它“觉得这个工具哪里别扭”。**

## Agent loop 的结构
示例的 agent loop 非常朴素：

- 初始化 messages
- 调 `client.messages.create(...)`
- 如果 stop_reason 是 `tool_use`
  - 找出 tool call
  - 执行本地函数
  - 记录工具耗时
  - 把 tool_result 塞回 messages
  - 继续下一轮
- 直到不再要工具

优点是简单直白，容易自己扩展。

它还顺手记录了每个工具：

- 调用次数
- 每次 duration

这让报告能统计：

- 每题调用了多少次工具
- 哪个工具被频繁调用
- 是否存在低效反复调用
- 工具本身是不是太慢

## Evaluation file 怎么设计
示例里用的是 XML evaluation file，大致结构就是每个 task 包：

- `<prompt>`
- `<response>`

也就是：

- prompt = 题目
- response = ground truth

然后通过 `parse_evaluation_file()` 读出来。

这个设计虽然简单，但很适合快速起步。你后面完全可以扩展成：

- 多个 grader
- 允许多答案
- 附带 metadata
- 指定预期工具
- 指定 task category

但 cookbook 想表达的是：
**先把最小可用 eval 跑起来，比一开始设计超复杂框架更重要。**

## 单任务评测时收集什么
`evaluate_single_task()` 对每个任务会收这些结果：

- `prompt`
- `expected`
- `actual`
- `score`
- `total_duration`
- `tool_calls`
- `num_tool_calls`
- `summary`
- `feedback`

这里很值得注意的是：
它不是只存一个 pass/fail，而是把 transcript 抽象成了一个“结构化诊断结果”。

这样你能看到：

- 答错是不是因为 tool 根本没被理解
- tool 被调用太多次，是不是 description 不清
- tool error 多，是不是 schema 不好
- answer 对了但路径很别扭，是不是还有优化空间

## 报告里看什么
示例 report 会给一层总览：

- Accuracy
- Average Task Duration
- Average Tool Calls per Task
- Total Tool Calls

然后每个 task 单独展开：

- Prompt
- Ground Truth Response
- Actual Response
- Correct / Incorrect
- Duration
- Tool Calls
- Summary
- Feedback

这其实已经够做第一轮工具优化了。

## 页面里的示例：一个故意很差的 calculator 工具
Cookbook 最有意思的地方是，它不是展示一个完美工具，而是故意放了一个很烂的 calculator 示例：

- `description` 是空的
- `expression` 参数说明也是空的
- 内部用 `eval()` 做计算
- 只支持很有限的表达式能力

然后拿 8 个数学题去跑 eval。

结果：

- Accuracy: 7/8（87.5%）
- Average Task Duration: 22.73s
- Average Tool Calls per Task: 7.75
- Total Tool Calls: 62

也就是说，虽然很多题最后能做对，但 agent 用这个工具的过程很痛苦，表现为：

- 调用次数很多
- 试错多
- 经常不知道支持什么语法
- 缺少常见函数
- 会在格式问题上翻车

## 这个示例暴露了哪些典型工具问题
从页面展示的反馈里，几个问题特别典型。

### 1. 工具描述太差
calculator 的 tool description 为空，parameter description 也为空。

导致 agent 不知道：

- 支持什么语法
- 指数是不是 `^` 还是 `**`
- 能不能 `round()`
- 能不能 `sqrt()`
- 能不能用 `math` 模块

这会直接把大量 token 浪费在试错上。

### 2. 工具能力和任务预期不匹配
作为 calculator，它却不支持很多常见数学能力：

- `round()`
- `int()`
- `sqrt()`
- 三角函数
- `math` 库

于是 agent 被迫：

- 手写替代公式
- 用幂运算模拟平方根
- 用近似值代替 `sin(45°)` / `cos(45°)`
- 在最后人工凑格式

这不是 agent 笨，是工具设计得别扭。

### 3. verifier 也可能过严
第一题实际答案是 `$11,614.72`，ground truth 是 `11614.72`，所以被判错。

这正好暴露了一个 eval 设计问题：
**如果 verifier 过于依赖 exact string match，就会把语义正确但格式不同的答案判成错。**

这跟 Anthropic 其他 eval 文章一脉相承：
尽量避免脆弱的 grading。

### 4. tool-call 数量是一个很有价值的信号
平均每题 7.75 次工具调用，不算低。

这通常说明几件事之一：

- 工具本身粒度不对
- 工具描述不清楚
- 错误信息不够指导性
- 返回内容没有帮助 agent 缩小搜索空间

也就是说：
**高准确率不代表工具已经好用。**
如果达到正确答案的代价很高，仍然值得优化。

## 这篇 cookbook 真正想传达的方法论
我觉得最值钱的不是示例代码本身，而是背后的评测思路：

### 1. 工具可以被独立评测
很多团队会把工具问题和模型问题、prompt 问题、workflow 问题混在一起。
这篇其实是在说：
**你可以把 tools 当成单独可优化对象，做独立 benchmark。**

### 2. agent 本身可以当“工具 UX 审查员”
通过 `<feedback>` 和 `<summary>`，agent 不只是执行者，还是：

- 工具说明审查员
- schema 可用性反馈器
- token inefficiency 观察员
- error message 可读性测试器

### 3. 不要只看 pass/fail
准确率很重要，但工具优化更应该同时看：

- 调用次数
- 耗时
- 报错率
- transcript 中的困惑点
- agent 的自然语言反馈

### 4. 先用简单 harness 跑起来
这里的实现很朴素，甚至用 `eval()` 调本地函数，明显是 demo 级别。
但这恰好说明：
**第一版工具评测，不需要重型框架。最关键的是尽快形成迭代闭环。**

## 我觉得最值得记的点

### 1. 工具评测的目标不是“能不能用”，而是“好不好用”
agent 最终做对，不等于工具设计合理。

### 2. 一个差工具会把 token 和时间悄悄烧掉
哪怕准确率还不错，trial-and-error 太多也说明有系统性浪费。

### 3. exact match verifier 很容易错杀
尤其是数值题和格式题，最好做更稳健的比较。

### 4. summary + feedback 是高价值训练信号
它们可以直接用来驱动下一轮 description / schema / error handling 改进。

### 5. 这是 evaluation-driven tool design 的最小可行闭环
任务集、简单 loop、结构化反馈、统计报告，四样就够启动。

## 一句话总结
**这篇 Claude Cookbook 展示了一个非常实用的 tool evaluation 模式：把工具从整个 agent 系统里单独抽出来，用一批真实任务、简单 agent loop、结构化 summary/feedback 和调用指标去评估工具的可用性与效率，再据此迭代优化工具描述、参数设计、能力边界和 verifier。**
