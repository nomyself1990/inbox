# Prompt Caching 作为 Harness 工程的一等约束

- **来源:** https://yage.ai/share/prompt-caching-harness-constraint-20260404.html
- **作者:** 未署名
- **日期:** 2026-04-04
- **出处:** yage.ai
---

## 概览
这篇文章讨论的是一个很容易被低估、但实际上会反向塑造整个 agent / harness 设计的底层约束：**prompt caching 不是“上线后再顺手优化一下”的小技巧，而是决定系统成本、延迟，甚至决定某些架构能否成立的 viability constraint。**

文章从 OpenClaw 一个看起来反直觉的 PR 讲起：在 context compaction 时，优先删除最新的 tool results，而不是最旧的历史。直觉上这很怪，因为最新上下文通常最相关；但从 cache economics 的角度看，这反而是更合理的选择——**前缀稳定性往往比局部上下文新鲜度更值钱。**

## 核心判断

### 1. Prompt caching 决定的是“能不能做”，不只是“做得快不快”
文章强调，agent 场景里常见的是 **input token 远大于 output token**。引用 Manus 的数据，平均 input/output 比大约是 **100:1**。

这意味着：

- 大部分成本花在反复处理长上下文上
- cache hit / miss 会直接决定成本基线
- 首 token 延迟也高度依赖缓存命中

文中引用多家供应商的数据：Anthropic、OpenAI、DeepSeek 的缓存命中价格都可能只有 miss 的一成左右。DeepSeek 还给出过一个例子：128K prompt 在高缓存命中下，首 token 延迟可从 **13 秒降到 500 毫秒**。

所以作者的判断很明确：

- 如果 cache miss rate 长期偏高，系统成本会被放大数倍
- 如果冷启动太慢，sub-agent、background agents、speculative execution 这些模式在交互上就不成立

一句话说：**prompt caching 约束的是系统架构的可行边界。**

### 2. Cache discipline 会反向塑造 harness design
因为缓存匹配依赖严格前缀比对，哪怕只改一个 token，从改动点往后的缓存都会失效。所以发给模型的前部内容——system prompt、tool definitions、早期 message——就不再是可以任意改写的数据，而是近似“半不可变”的稳定前缀。

这会连锁影响多个设计点：

- compaction 应该优先从尾部删，而不是从头部删
- tool definitions 必须确定性排序
- 不要频繁动态增删工具
- 图片和大型内容应尽量延迟裁剪，避免改写前缀
- sub-agent 的参数与 prompt 结构也要考虑缓存共享

作者的核心观点是：**很多看起来违背直觉的实现，本质上是在全局 cache economics 压过局部功能直觉之后形成的结果。**

### 3. 无法度量的 cache 问题，最终只会变成静默账单
Prompt caching 的坑在于：miss 不会报错，只会悄悄涨成本、拉高延迟。

文章建议至少建立三类观测：

- 每次 API 调用的 cache hit / miss / creation token 统计
- cache break 来源归因（system prompt 变化、tool list 顺序变化、历史消息被改写、compaction 等）
- 主 agent 与 sub-agent 分开统计缓存表现

作者还提到 Claude Code 泄露源码中的 `promptCacheBreakDetection.ts`，它会系统归因每一类 cache break 的来源。这个思路很值得学：**把模糊的“成本怎么又高了”变成可归因的工程问题。**

## 我觉得值得记的点

### 1. “删最新 tool result”这件事其实很合理
开头那个 PR 的逻辑支点很漂亮：

- 最新 tool result 信息密度高，但通常可以重新获取
- 早期前缀一旦被改坏，缓存只能全价重建

也就是说，二者并不对称。**可重取的数据，优先牺牲；不可廉价恢复的缓存前缀，优先保护。**

### 2. 工具列表稳定，比“按需灵活增删”更重要
文章提到如果 tool definitions 的序列化顺序不稳定，整段前缀缓存就会崩掉。Manus 的经验也类似：宁可保留工具位置，再通过 masking 控制当前可用性，也不要频繁改工具列表本身。

这个点很工程化，也很反直觉：**为了缓存稳定，动作空间管理最好通过约束解码来做，而不是通过频繁重写 prompt 前部来做。**

### 3. 大内容不要总想着往 context 塞，文件系统指针往往更优
文章在图片 pruning 和大型内容管理上给了一个重要启发：

- 如果你先把大对象塞进对话，再在中途裁剪它，等于先污染前缀、再破坏前缀
- 更好的做法是从一开始就把网页、PDF、长文写入文件系统，只在上下文里保留引用路径

这和 Manus 那套“把文件系统当 context extension”的思路是完全一致的。对于 agent 来说，这通常比盲目依赖长上下文更健康。

### 4. Sub-agent 很容易成为隐性成本放大器
这部分我觉得特别值得记。

主 agent 维护得再好的缓存，对短生命周期 sub-agent 往往也没法直接复用。每启动一个新子 agent，可能就是一次新的缓存冷启动。如果子任务又很短，缓存还没来得及复用，它就结束了。

文中还提到一个很细的例子：像 `reasoning_effort` 这种参数变化，也可能破坏缓存共享。表面上是“为了省点输出 token”，实际上可能因为 miss 让总成本更高。

这提醒一个很现实的事：**sub-agent 架构不是拆得越细越好，任务粒度必须和缓存复用一起设计。**

### 5. 这篇文章本质上是在谈“架构约束优先级”
它真正有价值的地方，不只是讲 prompt caching 本身，而是在提醒：

- 成本模型会反向决定系统形态
- 延迟模型会反向决定哪些交互模式可用
- 看起来局部更优的策略，在全局成本函数下可能反而更差

从这个角度看，prompt caching 不是 API 细节，而是 harness architecture 的一部分。

## 一句话总结
**这篇文章最值得记的结论是：在 agent / harness 系统里，prompt caching 应该被当成一等架构约束来设计。前缀稳定性、工具定义稳定性、上下文 append-only、sub-agent 粒度和可观测性，都是同一个底层约束向上游扩散后的工程结果。**
