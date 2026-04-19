# Anthropic：AI Agent 评测（evals）去魅，从 0 到 1 的实战路线图

- **来源:** https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents
- **作者:** Mikaela Grace、Jeremy Hadfield、Rodrigo Olivares、Jiri De Jonghe
- **日期:** 2026-01-09
- **出处:** Engineering at Anthropic
---

## 概览
这篇文章的核心不是“eval 很重要”这种空话，而是把 **agent eval 到底在评什么、怎么搭、怎么避免自欺欺人** 讲清楚了。

Anthropic 的主线很明确：**agent 越有用，就越难评测**。因为它不是单轮输出文本，而是会多轮推理、调工具、改环境状态、根据中间结果调整策略。所以评估不能只盯最终一句话，而要同时看：

- 任务定义是否清晰
- transcript / trajectory 里发生了什么
- 最终 outcome 是否真的达成
- grader 是否在测真正重要的东西
- harness 和环境有没有把结果搞脏

一句话概括：**评测 agent，本质上是在评“模型 + agent harness + 环境 + grader”的整个系统，而不是单独评某个模型。**

## 文章先做了哪些关键定义
Anthropic 先把一些经常被混着说的概念拆开了，这部分很值钱。

- **Task**：一条具体测试题，有输入和成功标准
- **Trial**：同一个 task 的一次尝试；因为输出有随机性，所以要跑多次
- **Grader**：评分逻辑；一个 task 可以有多个 grader
- **Transcript / Trace / Trajectory**：完整执行记录，包括工具调用、中间步骤、推理过程等
- **Outcome**：环境里的最终结果，不只是 agent 嘴上说“完成了”
- **Evaluation harness**：跑评测的基础设施，负责执行、记录、评分、聚合
- **Agent harness / scaffold**：把模型变成 agent 的那层系统，负责输入处理、工具编排、结果返回
- **Evaluation suite**：一组任务，通常围绕同一类能力或行为

这里最关键的区分是：**transcript 不等于 outcome，agent 说自己做成了也不算，得看环境里是不是真的成了。**

## 为什么要尽早做 eval
文章对这个问题的回答非常工程化：

早期团队通常靠这些东西推进：

- 手工测试
- 自己 dogfood
- 直觉判断

开始阶段这样没问题，但一旦 agent 上线、用户变多、迭代频繁，没有 eval 很快会进入一种很典型的坏循环：

- 用户说“感觉变差了”
- 团队只能靠猜
- 修一个 bug，又引入别的问题
- 分不清是真的退化，还是噪音
- 也没法在发版前把几百种场景自动回归一遍

Anthropic 的判断是：**eval 的价值是复利型的。**
前期写它像成本，后期它会变成整个团队的加速器。

文章给了几个特别实际的收益：

- 把“成功标准”写清楚，减少团队理解偏差
- 把真实失败案例变成可复现测试
- 让模型升级更快，不用每次都靠人工大回归
- 顺手获得 latency / token / cost / error rate 等长期基线
- 让 product 和 research 有共同可优化的指标

## 三类 grader：代码、模型、人
Anthropic 把 grader 分成三大类，而且强调 **agent eval 通常要混搭**。

### 1. Code-based graders
典型做法：

- 字符串匹配 / regex / fuzzy match
- 单元测试 / fail-to-pass / pass-to-pass
- 静态分析（lint / type / security）
- outcome verification
- 工具调用检查
- transcript 分析（turn 数、token 数等）

优点：

- 快
- 便宜
- 客观
- 可复现
- 好调试

缺点：

- 容易太脆
- 对合理但不在预期里的解法不友好
- 很难覆盖主观质量

### 2. Model-based graders
典型做法：

- rubric 打分
- 自然语言断言
- pairwise comparison
- reference-based evaluation
- multi-judge consensus

优点：

- 灵活
- 可扩展
- 能处理开放式任务
- 能捕捉细微质量差异

缺点：

- 非确定性
- 比 code grader 贵
- 必须经常拿人类标准校准

### 3. Human graders
典型做法：

- SME review
- crowd judgment
- 抽样 spot-check
- A/B testing
- inter-annotator agreement

优点：

- 质量最高
- 最接近真实用户/专家判断
- 可以拿来给 LLM grader 校准

缺点：

- 慢
- 贵
- 很难高频大规模使用

文章的真实建议不是“选最先进的”，而是：
**能用 deterministic 的地方先用 deterministic；必须处理开放性和主观质量时，再上 model-based；human 用于校准和抽查。**

## Capability eval 和 regression eval 不是一回事
这是文章里一个很重要但常被混淆的点。

### Capability / quality eval
问的是：
**这个 agent 现在到底能把哪些事做好？**

它应该有一定难度，最好一开始 pass rate 不高，让团队能看到爬坡空间。

### Regression eval
问的是：
**这个 agent 以前会做的事，现在还会不会？**

它应该接近 100% pass，用来防止回退。

Anthropic 的建议很像软件测试里的分层：

- capability suite 用来推动能力上限
- regression suite 用来守住已达到的下限

随着某批 capability tasks 逐渐变成高通过率，它们就可以“毕业”成 regression suite。

## 不同 agent 类型怎么评
文章按四类 agent 讲了评测思路。

### 1. Coding agents
核心是：**任务要清晰，环境要稳定，验证要尽量 deterministic。**

典型 grader：

- 单元测试 / 集成测试
- 静态分析
- state check
- transcript metrics
- 少量 LLM rubric 看代码质量或交互行为

Anthropic 这里强调了一个经验：
**对 coding agent，通常 correctness 先靠 tests，quality 再靠 rubric。**
别一开始就把复杂 grader 堆满。

### 2. Conversational agents
难点在于：
**交互质量本身就是被评对象的一部分。**

所以往往要同时看：

- 最终 state 有没有正确变化
- turn 数有没有失控
- tone 是否合适
- 工具使用是否 grounded

而且很多时候需要用第二个 LLM 来模拟用户，把多轮对话跑出来。

### 3. Research agents
这是最难标准化的一类。

因为“全面”“可靠”“足够好”高度依赖场景，不能像 coding 一样只看 tests pass 不 pass。

文章给出的可行做法是组合多个 grader：

- groundedness：结论有没有被来源支持
- coverage：关键点有没有覆盖
- source quality：引用的是不是权威来源
- exact match：对客观事实题仍然有用
- LLM rubric：看综合质量、连贯性、完整性

关键提醒：**research 评测里的 LLM grader 必须持续和专家判断对齐。**

### 4. Computer use agents
这种 agent 在 GUI 里点点点、输入、滚动、截图，用的是和人一样的界面。

因此评测要跑在真实或沙箱环境里，然后检查：

- 页面/URL/元素状态是否对
- 后端状态是否真的改了
- 文件/配置/数据库是否真的达到目标

Anthropic 还提了一个很实用的点：
**browser use 里 DOM 方式和 screenshot 方式各有成本，评测也可以专门检查 agent 是否选对了交互方式。**

## 非确定性：pass@k 和 pass^k 要分开看
这篇文章把 agent eval 里最容易被忽略的问题讲透了：
**agent 是随机系统，同一个 task 这次过了，下次可能不过。**

所以不能只看“这次跑通没”。

### pass@k
表示：跑 k 次，**至少有一次成功** 的概率。

适合看这类场景：

- 允许多次尝试
- 只要找到一个可行解就行
- 比如某些 coding / search / planning 场景

### pass^k
表示：跑 k 次，**每一次都成功** 的概率。

适合看这类场景：

- 用户每次都希望稳定成功
- 一次失败就影响体验
- 比如 customer-facing agent

Anthropic 用这两个指标提醒大家：
**同一个 agent，可能 pass@10 很高，但 pass^10 很低。**
也就是说，它“偶尔很强”，不代表“持续可靠”。

## 从 0 到 1 的 8 步路线图
这是整篇最实操的部分。

### Step 0. 早点开始
别等 200 个题库。**20–50 个真实 task 就够启动。**

### Step 1. 从你现在手工在测的东西开始
把这些转成测试：

- 发版前手工 check 的行为
- 用户常做的关键路径
- bug tracker / support queue 里的真实失败

### Step 2. 任务必须足够明确，并且要有 reference solution
判断标准很简单：
**两个领域专家会不会独立得出同样的 pass/fail 结论？**

如果不会，任务定义或 grader 就有问题。

Anthropic 还特别提醒：
- 任务要求里没写清楚的，grader 不该默认要求
- 0% pass@100 往往不是模型太差，而是 task/grade 坏了
- 每个 task 最好都配一个已知能通过的 reference solution

### Step 3. 构建平衡题集
不能只测“该发生时会不会发生”，还要测“**不该发生时能不能克制住**”。

比如只测“该搜时会不会搜”，就会把 agent 训练成“逢题必搜”。

### Step 4. 搭稳定的 eval harness 和隔离环境
每次 trial 都应该从干净环境起步，避免：

- 前一次运行残留状态
- cache 污染
- 资源不足带来的相关性故障
- agent 利用历史痕迹“作弊”

文章举了个很典型的例子：Claude 会去翻之前 trial 的 git history，从而获得不公平优势。

### Step 5. 认真设计 grader
这一段信息量很大，核心观点有几个：

- 尽量评 **结果**，不要死盯具体步骤
- 太细的 tool-call 顺序检查非常脆
- 多组件任务应该允许部分得分
- LLM-as-judge 要和人工持续校准
- 给 judge 留 “Unknown” 出口，避免硬编
- 多维度任务最好拆成多个维度分别评，而不是一个大 judge 包办
- grader 要防绕过、防 exploit

Anthropic 还举了几个很有警示意义的例子：

- CORE-Bench 上 Opus 4.5 一开始只有 42%，排查后发现是 grader 太僵、task 含糊、部分题不可复现，修完后到 95%
- METR 的 benchmark 里有任务“文字要求”和“打分门槛”不一致，结果老实按要求做的模型反而吃亏

这部分最值得记的一句话是：
**低分不一定说明 agent 差，可能是 eval 在惩罚正确行为。**

### Step 6. 读 transcript
Anthropic 说得很直白：**不读 transcript，你根本不知道 grader 是否靠谱。**

当任务失败时，必须能回答：

- agent 真的犯错了吗？
- 还是 grader 错杀了有效解？
- harness / 环境有没有引入噪音？

### Step 7. 监控 capability eval saturation
当某套 eval 接近 100%，它就不再适合衡量进步，只适合做 regression。

如果题太旧、太浅，会让能力提升看起来像没提升。

### Step 8. 长期维护，把 eval 当活资产
Anthropic 内部实践是：

- 核心 infra 由专门 eval 团队维护
- 具体 task 更多由产品团队和领域专家贡献
- product 团队应该像维护单元测试一样维护 eval

文章还强调一个我很认同的观点：
**离用户和需求最近的人，往往最适合定义“成功长什么样”。**
不仅研究员能写 eval，产品经理、客服、销售也可以贡献任务。

## 自动 eval 不是全部，它只是“多层奶酪”里的一层
文章最后给了一个很成熟的框架：
真正理解 agent 质量，不能只靠 automated eval，还要结合：

- production monitoring
- A/B testing
- user feedback
- manual transcript review
- systematic human studies

它借用了 Swiss Cheese Model：
**没有任何一层能挡住所有问题，但多层组合能显著降低漏网率。**

这点非常重要，因为很多团队会把 eval 神化成“唯一真相”。Anthropic 的态度反而更稳：
**eval 是开发加速器和回归守门员，但不是现实世界的全部。**

## 我觉得最值得记的点

### 1. 评 agent，不是在评一句答案，而是在评整个系统
模型、harness、工具、环境、grader 全都影响结果。

### 2. transcript 和 outcome 要分开看
说做成了不算，环境里真的做成才算。

### 3. grading 最容易错在“太刚性”
如果你过度规定步骤，就会惩罚那些同样正确、甚至更聪明的解法。

### 4. pass@k 高，不代表可上线
很多 agent 是“偶尔很强”，但还不够稳定。面向用户时更该盯 pass^k。

### 5. 低分可能是 eval 坏了，不一定是 agent 坏了
任务歧义、grader 僵化、环境脏、harness 受限，都会把分数拉低。

### 6. 早点写 eval，不是拖慢开发，而是让需求更清楚
eval 不只是测系统，也是逼团队定义“到底什么算成功”。

## 一句话总结
**Anthropic 这篇文章把 agent eval 讲得非常落地：从任务设计、grader 选型、非确定性指标、环境隔离到长期维护，核心原则都是别迷信单一分数，评结果而不是死抠路径，尽早把真实失败转成可复现测试，并把 eval 当成持续演化的工程系统。**
