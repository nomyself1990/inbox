# Context Engineering for AI Agents：从 Manus 的实践里学什么

- **来源:** https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus
- **作者:** Yichao "Peak" Ji
- **日期:** 2025-07-18
- **出处:** Manus Blog
---

## 概览
这篇文章的核心判断很鲜明：**在 agent 产品里，context engineering 不是 prompt 小技巧，而是系统设计本身。**

Manus 团队一开始就面临一个选择：是训练端到端 agent model，还是站在 frontier models 的 in-context learning 能力之上做系统。最后他们选择了后者，原因很现实——反馈回路更快、迭代成本更低，而且产品能力能随着底层模型进步一起上涨。

作者把这套方法叫 context engineering：不是只写 prompt，而是系统性地设计上下文的组织方式、缓存策略、工具空间、外部记忆、错误保留和注意力控制机制。

一句话总结：**agent 的上限不只取决于模型有多强，还取决于你怎么塑造它每一步看到的上下文。**

## 核心判断

### 1. KV-cache 命中率，是生产级 agent 的头号指标之一
文章开篇就给了一个非常工程化的观点：如果只能盯一个指标，作者会选 **KV-cache hit rate**。

原因很直接：agent 的上下文会随着每一轮 action / observation 不断增长，但每一步输出通常很短，经常只是一个函数调用。于是 agent 场景里的 input/output token 比例会极度失衡。文中给的数据是：在 Manus 里，平均输入输出 token 比大约是 **100:1**。

这意味着：
- prefilling 成本比 decoding 更关键
- 稳定复用前缀缓存，会直接影响延迟和价格
- cache 命中不好，agent 越长越贵、越慢

作者还举了 Claude Sonnet 的例子：cached input token 和 uncached input token 的价格差到 **10x**。所以这不是微优化，是一等公民级别的优化。

### 2. 稳定前缀、append-only、确定性序列化，是 context 设计基本功
为了提高 KV-cache 命中率，文中提了几条非常实操的原则：

- **保持 prompt prefix 稳定**：别在系统提示最前面塞“精确到秒”的时间戳，这种单 token 变化都会让缓存从那个位置后全部失效。
- **上下文尽量 append-only**：不要回头改之前的 action / observation。
- **序列化必须确定性**：JSON key 顺序不稳定，可能悄悄打爆 cache。
- **必要时显式设置 cache breakpoint**：有些推理框架不会自动做增量 prefix caching。

这个部分很像在说：**agent 不是只要会推理，还要能便宜地持续推理。**

## 工具空间怎么设计

### 1. 工具不是越多越强，动作空间太大反而会把 agent 搞笨
作者点了一个很多人做 agent 时会踩的坑：随着能力扩展，tool count 会膨胀，尤其 MCP 一接进来，很容易一下多出一大堆工具。工具一多，模型更容易：
- 选错动作
- 走低效路径
- 产生 schema violation
- 幻觉出不存在的 action

这其实是在提醒：**动作空间本身就是认知负担。**

### 2. 不要在中途动态增删工具，优先 mask，不要 remove
他们尝试过动态 action space，比如按需加载工具，但最终经验是：**除非绝对必要，不要在 iteration 中途增删工具定义。**

原因有两个：
1. tool definitions 通常位于上下文前部，任何变化都会破坏 KV-cache；
2. 前面的轨迹还在引用旧工具，后面工具定义却变了，会让模型困惑。

Manus 的解法不是 remove tools，而是借助一个 context-aware state machine，在 decoding 阶段 **mask token logits**，限制模型只能从某一类工具里选。

这个思路很妙：
- 工具定义保持稳定
- cache 不被频繁打爆
- 动作空间仍能按状态约束

### 3. 工具命名最好有前缀语义
文章提到他们故意把工具按前缀命名，比如：
- `browser_` 开头是浏览器相关
- `shell_` 开头是命令行相关

这样做之后，就能配合 response prefill / constrained calling，更容易在某个状态下只允许一组工具。

这算是个很小但很实用的工程细节：**命名规范本身也会影响 agent 的可控性。**

## 把文件系统当成上下文

### 1. 超长上下文不是万能药，很多时候反而是负担
文中对长上下文的批评很到位。就算 frontier model 给了 128K+ context，也还是有三个现实问题：
- observation 可能大得离谱，比如网页、PDF
- 上下文太长后模型表现会下降
- 长输入本身就很贵

所以问题不是“窗口够不够长”，而是 **什么信息应该留在 context，什么信息应该外化**。

### 2. Manus 把文件系统视为 ultimate context
这是全文我觉得最值得记的一点之一：**file system is the ultimate context**。

原因是：
- 容量几乎无限
- 天生持久化
- agent 自己就能读写
- 可以当作结构化的外部记忆

也就是说，Manus 不试图把所有状态都塞进 token window，而是让模型学会：
- 需要时把重要中间结果写文件
- 需要时再从文件里读回来

这跟很多优秀 agent 系统的方向是一致的：**把状态外化到环境，而不是硬塞进上下文。**

### 3. 压缩策略必须可恢复，不要做不可逆压缩
文章强调，他们的压缩策略都尽量是 **restorable** 的。

比如：
- 网页正文可以从上下文里删掉，只要 URL 还在
- 文档全文可以不放在上下文里，只要 sandbox 里的路径还在

这个原则很关键，因为它避免了“为了省 token 把关键线索永久丢掉”。

## 如何操纵注意力

### 1. todo.md 不是装可爱，是 recitation 机制
如果你用过 Manus，可能会发现它喜欢生成 `todo.md`，然后一步步更新。文章解释说，这不是拟人化小把戏，而是有意为之的 **attention manipulation**。

典型任务平均大约有 **50 次 tool call**，这么长的 loop 很容易让模型：
- 偏题
- 忘记最初目标
- 被中间 observation 带跑

而不断重写 todo list，本质上是在 **把全局目标反复 recite 到上下文末尾**，让模型的近期注意力持续对准任务目标，减少 lost-in-the-middle 和 goal drift。

### 2. 自然语言可以当成轻量级控制机制
这个点挺有意思：他们没有上特别复杂的新架构，而是用自然语言 recitation 来偏置模型关注点。

换句话说，**不是所有控制问题都要靠模型结构升级，有些可以靠上下文编排解决。**

## 错误要不要隐藏

### 1. 保留错误轨迹，比“擦干净重来”更有价值
文章对 agent error handling 的观点很明确：**不要总想着把错误藏起来。**

常见冲动是：
- 重试
- 清理轨迹
- reset state
- 让模型像没犯过错一样继续

但 Manus 认为，这样做会抹掉关键信号。失败本身就是证据。模型看到失败动作、报错信息、stack trace 后，会隐式更新对环境的判断，降低重复犯错概率。

### 2. error recovery 是 agent 性能的重要标志
他们甚至认为，错误恢复能力是最能体现 agent 性的指标之一。但很多 benchmark 更关注理想条件下的一次成功，低估了 agent 在真实环境里处理失败的能力。

这个判断我很认同：**真实 agent 系统不是看“会不会错”，而是看“错了以后能不能学着绕开同样的坑”。**

## Few-shot 也会害人

### 1. 过于统一的上下文，会把 agent few-shot 到僵化
文章最后一个点也很妙：few-shot 在 agent 里不总是好事。

原因是模型很会模仿。如果上下文里充满高度相似的 action-observation pattern，它会机械地复制这个节奏，即使当前情况已经不适合。作者举的例子是批量看 20 份简历时，agent 容易进入重复动作惯性，最后 drift、过拟合甚至 hallucinate。

### 2. 需要有控制的多样性
Manus 的做法是人为引入一点结构化变化，比如：
- 不同 serialization template
- 不同措辞
- 少量顺序或格式扰动

这种受控噪声能打破模式固化，避免 agent 被自己 few-shot 进死胡同。

这个结论挺反直觉，但很有启发：**上下文太整齐，未必更好；适度多样性有时更稳。**

## 我认为最值得抄的几点

### 1. 把 KV-cache 命中率当成核心经营指标
如果 agent 是多步长链路、重工具调用场景，这几乎应该直接进 dashboard。

### 2. 工具空间稳定，比动态花活更重要
能 mask 就别删定义；能约束动作空间就别频繁改工具集合。

### 3. 用文件系统做外部记忆
尤其适合长任务、文档处理、多轮分析、代码代理这种 observation 巨大的场景。

### 4. 用 recitation 机制对抗注意力漂移
`todo.md`、plan.md、current_goal.md 这类东西，不只是给人看，也是给模型看。

### 5. 失败轨迹别轻易擦除
保留错误，是在给下一步决策提供反证数据。

### 6. 小心“被自己的 few-shot 绑架”
重复模式太强，会让 agent 越来越机械。

## 一句话总结
这篇文章最核心的价值，是把“context engineering”从一个模糊概念落到了具体工程原则上：**稳定前缀保缓存、用 mask 控动作、把文件系统当外部记忆、靠 recitation 拉回注意力、保留错误轨迹、避免被单一模式 few-shot。**

本质上，Manus 想表达的是：**agent 不是只靠模型智力取胜，更多时候靠你如何组织它的上下文与环境。**
