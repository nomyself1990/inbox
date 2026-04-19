# Anthropic：怎么给 AI Agent 写更有效的工具，以及如何用 Agent 反过来优化工具

- **来源:** https://www.anthropic.com/engineering/writing-tools-for-agents
- **作者:** Ken Aizawa（并有 Anthropic 多团队成员贡献）
- **日期:** 2025-09-11
- **出处:** Engineering at Anthropic
---

## 概览
这篇文章的核心观点很直接：**agent 的上限，很大程度上取决于它拿到的工具质量。**

Anthropic 不只是讲“tool description 要写清楚”这种浅层建议，而是给了一套完整流程：

1. 先快速做工具原型
2. 用真实任务做系统化 eval
3. 让 Claude 之类的 agent 读 eval transcript，反过来帮你改工具
4. 在迭代里沉淀工具设计原则

一句话总结：**工具不是给 deterministic software 用的 API 包装，而是给 non-deterministic agent 用的行动界面；因此设计原则、验证方式、优化方法都要重写。**

## 文章先讲清了一个前提：tool 不是普通 function
Anthropic 先区分了两类系统：

- **deterministic system**：相同输入得到相同输出
- **non-deterministic agent**：相同起点也可能走不同路径、给不同答案

传统函数像 `getWeather("NYC")`，你知道每次都会怎么跑。
但 agent 面对“今天要不要带伞”时，可能：

- 直接调用天气工具
- 先问你在哪
- 靠已有知识先答一个大概
- 或者干脆误用工具/不用工具

所以工具设计不该再按“给工程师调用 API”的思路来写，而要按“**让 agent 更容易理解、选择和正确使用**”的思路来写。

## 他们给出的实战流程

### 1. 先搭一个可跑的 prototype
建议不要先空想什么工具最优，而是：

- 先做原型
- 挂到本地 MCP server 或 DXT
- 接进 Claude Code / Claude Desktop / API
- 自己试一轮
- 用真实用户反馈建立直觉

文章强调：**很多 tool ergonomics 问题，不上手跑是想不出来的。**

### 2. 给工具做 eval
然后不是靠感觉改，而是跑评测。

Anthropic 推荐：

- 用真实任务生成 evaluation tasks
- 每个 task 配一个可验证的结果或 outcome
- verifier 可以从 exact match 到让 Claude 当 judge
- 不要把 verifier 写得太死，别因为格式差异把正确答案打成错

他们特别强调：**强任务应该接近真实工作流，通常要多次 tool calls，甚至几十次。**

文中给了几个对比：

更强的任务像：
- 帮 Jane 下周约会，附上上次会议笔记，还要订会议室
- 查某客户三次扣费是否波及其他客户
- 对要流失的客户做 retention offer，并分析原因、方案、风险

更弱的任务像：
- 给 jane@acme.corp 约个会
- 搜 customer_id=9182 的日志
- 找 customer ID 45892 的取消请求

意思很明确：**太浅的 sandbox task 测不出工具设计的真实问题。**

### 3. 用简单 agent loop 跑评测
Anthropic 推荐评测时直接用 API 跑最简单的 agent loop：

- 一个 task 一个 loop
- 给 agent task prompt + 工具
- 循环执行 LLM 调用和 tool calls

还建议在 system prompt 里让 agent 输出：

- structured response
- reasoning
- feedback

这样更方便分析它为什么会/不会调某个工具。

如果用 Claude，还可以开 interleaved thinking。

### 4. 除了 accuracy，还要看行为指标
除了顶层正确率，还建议统计：

- 每次 tool call 的 runtime
- 每个 task 总运行时间
- tool 调用次数
- token 消耗
- tool errors

因为这些指标能暴露很多结构性问题：

- 调很多冗余工具 → 可能分页、limit、返回格式有问题
- invalid parameter error 很多 → 描述或 schema 不清楚
- token 爆炸 → response 太啰嗦、工具粒度不对

Anthropic 提了一个很典型的例子：Claude 会在 web search query 后面无脑加 `2025`，导致搜索结果偏掉。后来他们通过修改 tool description 把模型往正确方向引导了。

### 5. 直接让 agent 帮你分析 transcript、重构工具
这部分很有意思，也很 Anthropic：

- 把 eval transcript 拼起来
- 贴进 Claude Code
- 让 Claude 帮你看哪里不一致、哪里低效、哪里 description 自相矛盾
- 甚至一次性帮你重构很多工具

文章直接说：**这篇文章里很多经验，本身就是他们反复用 Claude Code 优化内部工具实现后总结出来的。**

## 五个最重要的工具设计原则

### 1. 不是工具越多越好，要选对工具
这是全文最重要的原则之一。

很多人会犯的错是：
**把已有 API endpoint 原样包一层，觉得这就叫给 agent 提供工具。**

Anthropic 说这常常是错的，因为 agent 的 affordance 跟传统软件完全不同。

文章举了一个很好懂的例子：

如果你给 agent 一个 `list_contacts`，返回所有联系人，让它自己在上下文里一条一条看，那就是在浪费 context。
更好的做法是：

- `search_contacts`
- `message_contact`

也就是直接贴合高频工作流，把“中间计算”尽量下沉到工具内部。

文中的推荐模式是：

- 别拆成一堆低层原语
- 优先做少量高价值、高工作流贴合度的工具
- 工具可以在内部包多步逻辑，对外暴露一个更自然的 action

给出的例子包括：

- 与其 `list_users + list_events + create_event`，不如直接 `schedule_event`
- 与其 `read_logs`，不如 `search_logs`
- 与其 `get_customer_by_id + list_transactions + list_notes`，不如 `get_customer_context`

核心思想就是：
**工具应该帮 agent 减少上下文消耗，而不是把原始复杂度原封不动甩给 agent。**

### 2. 用 namespacing 帮 agent 建清楚边界
当 agent 同时接几十个 MCP servers、几百个 tools 时，命名冲突和功能重叠会让它很容易懵。

Anthropic 推荐用 namespace：

- 按服务：`asana_search`, `jira_search`
- 按资源：`asana_projects_search`, `asana_users_search`

他们还提到一个细节：
**前缀式还是后缀式命名，对不同 LLM 的影响可能不一样。**
所以别拍脑袋，最好靠 eval 决定。

这个点背后其实是在说：
工具命名本身就是 prompt engineering 的一部分。

### 3. 工具返回的信息要“高信号”，少低价值技术细节
Anthropic 的建议很明确：
**默认返回 agent 真正拿来继续行动的信息，而不是工程实现细节。**

比如优先返回：

- `name`
- `image_url`
- `file_type`

而不是优先塞：

- `uuid`
- `256px_image_url`
- `mime_type`

他们还指出：
agent 通常更擅长处理自然语言名词、语义化标识，而不是随机 UUID。
甚至只是把复杂 ID 换成更语义化的文本，或者简单 0-indexed ID，都能明显减少 hallucination、提高 retrieval precision。

但有时候 agent 又确实需要 technical IDs 去做下游 tool call。Anthropic 提供了一个实用方案：
**在工具里加 `response_format` 之类的 enum，让 agent 自己选 `concise` 或 `detailed`。**

这样：

- 想省 token 时返回精简版
- 需要继续链式调用时再拿详细版

他们给的例子里，concise response 只用了 detailed response 大约三分之一的 token。

### 4. 工具响应要为 token efficiency 优化
这一点跟上面一脉相承，但更偏系统优化。

对于可能返回很多内容的工具，Anthropic 建议组合使用：

- pagination
- range selection
- filtering
- truncation
- 合理默认参数

并且如果你做了 truncation，就别只说“truncated”。
而是要用返回内容去引导 agent：

- 更小范围搜
- 用 filter
- 分页拉
- 不要一把梭全量抓

同样，error message 也别只扔 opaque error code。
应该返回：

- 哪个参数有问题
- 为什么错
- 合法示例长什么样
- 下一步建议怎么改

也就是说：
**tool response 不只是数据返回，也是对 agent 的在线 steering。**

### 5. tool description 和 spec 本身就是 prompt
这是文章点得最狠的一点。

Anthropic 认为，最有效的优化方法之一就是：
**认真 prompt-engineer 你的 tool descriptions 和 specs。**

因为这些东西会直接进 agent context，它们会整体决定 agent 的 tool use 行为。

文中建议的写法很像“给新同事讲工具”：

- 把默认隐含的上下文写出来
- 解释特殊 query 格式
- 解释术语含义
- 解释资源之间关系
- 输入输出都要清楚
- 参数命名要避免歧义

比如别用：
- `user`

更好用：
- `user_id`

Anthropic 还说，小改 description 有时会带来非常大的收益。
他们提到 Claude Sonnet 3.5 在 SWE-bench Verified 上的 SOTA 提升，就有一部分来自对工具描述的精细修改。

## 我觉得这篇最值钱的点

### 1. 工具设计本质上是“把 agent 的认知负担往系统外搬”
你不是单纯在暴露 API，而是在决定：

- 哪些信息该由工具内部消化
- 哪些信息值得消耗 context 返回给 agent
- 哪些行动单元对 agent 来说最自然

### 2. tool description 不只是文档，而是 agent policy 的一部分
对 agent 来说，description、schema、error message、response format，全都在塑造行为。

### 3. eval 不是可选项，而是唯一靠谱的优化闭环
没有 eval，你只能靠感觉说“这个工具应该更好了”。
有了 eval，才能系统比较：

- 命名是不是更好
- 分页是不是更合理
- description 改完是否真有收益
- tool 粒度是否更贴工作流

### 4. 最好的工具往往不是底层 primitives，而是 workflow-native actions
这和很多 agent system 设计经验一致：
agent 不怕工具少，怕工具碎、重叠、啰嗦、低信号。

### 5. 可以让 agent 帮你优化 agent 的工具
这其实是个非常实用的方法论：
**把 transcript 作为训练数据 / 反馈数据，把 Claude Code 当 tool UX reviewer + refactoring engine。**

## 一句话总结
**Anthropic 这篇文章的核心结论是：给 agent 写工具时，不要把现有 API 生硬包一层，而要围绕真实工作流重新设计工具边界、命名、返回格式和描述；再通过 evaluation-driven 的闭环，用 agent 自己分析 transcript 并持续优化工具，从而提升整体 agent performance。**
