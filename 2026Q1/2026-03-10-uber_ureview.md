# uReview：Uber 大规模 AI 代码审查系统

- **来源:** https://www.uber.com/en-HK/blog/ureview/
- **作者:** Sonal Mahajan, Shauvik Roy Choudhary, Akshay Utture, Will Bond, Joseph Wang
- **出处:** Uber Engineering Blog
---

## 概览
uReview 是 Uber 自研的 AI 代码审查平台，定位是人类 reviewer 的辅助，负责识别功能性 bug、安全漏洞和编码规范问题。目前覆盖 Uber 六大 monorepo（Go、Java、Android、iOS、TypeScript、Python），处理每周约 65,000 个 diff 中的 90% 以上。

## 核心指标
- 75% 的评论被工程师标记为有用
- 65% 的评论在同一变更集中被采纳（对比人类 reviewer 仅 51%）
- 每周分析 10,000+ 次提交，中位延迟 4 分钟
- 每周节省约 1,500 开发者小时，折合约 39 开发者年/年

## 系统架构：四阶段流水线

### 阶段一：摄入与预处理
过滤低信号目标（配置文件、生成代码），构建结构化 prompt，包含周围函数、类定义、import 等上下文信息。

### 阶段二：评论生成
三个可插拔的专用 assistant 独立运行：
- **Standard Assistant：** 检测 bug 和逻辑缺陷
- **Best Practices Assistant：** 执行 Uber 内部编码规范
- **AppSec Assistant：** 识别应用层安全漏洞

### 阶段三：后处理与质量过滤
多层过滤机制：
- 通过定制 prompt 进行二次置信度评分
- 语义相似度过滤，合并重叠建议
- 类别分类器，基于历史数据抑制低价值评论类别

### 阶段四：投递与反馈
评论直接内联到代码审查平台，附带 Useful/Not Useful 评分链接。元数据通过 Kafka 流入 Apache Hive 进行分析和改进追踪。

## 模型评估
在标注了 ground-truth 问题的基准提交集上，按 precision、recall、F1 评估：
- **最优配置：** Anthropic Claude 4 Sonnet（生成器）+ OpenAI o4-mini-high（评分器）
- **次优配置：** Claude 4 Sonnet + OpenAI GPT-4.1（F1 低 4.5 个点）
- 测试过的其他模型包括 GPT-4.1、O3、O1、Meta Llama-4、DeepSeek R1

## 评估方法论
**自动评估：** 在最终提交上重跑 uReview 5 次。如果所有重跑都未再产生语义相似的建议，则判定该评论已被修复。通过多次运行消除 LLM 的随机性。
**手动评估：** 维护一组人工标注的基准数据集，在部署前进行本地迭代验证。

## 为什么自建而非用第三方工具
三个原因：
- **平台限制：** 大多数第三方工具依赖 GitHub，Uber 使用 Phabricator
- **质量问题：** 评估过的第三方工具假阳性率高、有价值的真阳性少、集成能力有限
- **成本效率：** 在 Uber 的规模下，uReview 的运营成本比第三方工具低一个数量级

## 关键经验
**精度优先于数量。** 低置信度的建议会侵蚀用户信任，高质量反馈比评论数量重要得多。
**反馈闭环至关重要。** 在每条评论中嵌入简单评分链接，结合自动化评估，大规模收集反馈模式。
**多阶段 chaining 比单次 prompting 有效得多。** 后处理和质量过滤的重要性超过生成阶段本身。
**评论类别选择决定用户接受度。** 开发者拒绝可读性和风格类建议，重视正确性 bug、缺失的错误处理、最佳实践违规。
**纯代码分析的局限。** uReview 无法访问 PR 描述、feature flag、schema 和文档，限制了系统设计层面的评估能力。
**渐进式发布配合数据驱动迭代。** 分阶段部署，配合 precision-recall 仪表盘，实现快速修复和数据驱动的迭代。
**CI 阶段部署优于 IDE 集成。** 在代码审查平台上的 CI 阶段部署比 IDE 内置更能保证检查的执行率。
**LLM 与 Linter 各司其职。** 简单的语法模式适合 linter，语义理解（如识别表示时间的整数变量）适合 LLM。

## 未来方向
扩展上下文丰富度，覆盖更多审查类别（性能、测试覆盖率），开发面向 reviewer 的代码理解和风险识别工具。核心原则是保持工程师的控制权。
