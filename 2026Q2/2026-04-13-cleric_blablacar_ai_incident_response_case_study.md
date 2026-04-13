# Cleric × BlaBlaCar：用 AI 改造事故响应

- **来源:** https://cleric.ai/resources/case-studies/how-the-worlds-leading-community-based-travel-network-is-transforming-incident-response-with-ai
- **作者:** 未署名
- **日期:** 未注明
- **出处:** Cleric Case Study
---

## 概览
这是一篇 Cleric 官方案例，讲的是 **BlaBlaCar** 怎么把 AI 引入事故响应流程，目标不是“替代 SRE”，而是把一部分重复、低层级、跨系统排查工作自动化，让 SRE 和工程师更快收敛根因。

如果只记一句话：**BlaBlaCar 把 Cleric 当成一个能在 Slack 里做一级排障的 AI SRE companion，用来缩短告警调查时间、减少 root cause 定位成本，并把资深 SRE 的经验向更多团队扩散。**

## 背景
BlaBlaCar 是一个跨 21 个国家的多模式出行平台，业务包括：

- 长途拼车
- 自营城际大巴
- 巴士 / 火车票务市场
- 短途 ridesharing

它的可靠性体系并不小，但 SRE 团队本身很精干：

- **5 人 SRE 团队**
- 归属一个 **40 人的 Foundations 组织**
- 支撑 **200+ 工程师**
- 运行在 **Kubernetes + Istio service mesh** 上
- 每天支持 **200+ 次 CI/CD 部署**
- 以统一可观测性和 feature-level SLO 为基础

## 他们遇到的核心问题
文章里最真实的点，不是“告警太多”，而是 **症状型告警很多，但根因常常在别的服务里**。

几个关键痛点：

- **高优先级事故 MTTR 接近 2 小时**
- 多个团队可能同时看到“自己有问题”，但真正的源头在依赖项里
- 月度 review 能看到趋势，但很难快速提炼出高价值 action item
- 团队间 **Kubernetes 成熟度不均衡**，知识分布不平均

文章里反复强调一个问题：**真正难的不是看到 alert，而是快速 scope 问题、判断影响面、定位最初根因。**

## 他们想要的，不只是省时间
BlaBlaCar 想验证的其实有三件事：

- **Recovery speed:** 能不能把重复调查自动化，缩短恢复时间
- **Cross-service intelligence:** 能不能跨服务自动关联信号，更快暴露 root cause
- **Expertise multiplication:** 能不能把资深工程师 / SRE 的诊断能力扩散给所有团队

这个 framing 我觉得很对：**不是把 AI 当“便宜劳动力”，而是当“经验放大器”和“跨系统调查器”。**

## 为什么选 Cleric
BlaBlaCar 选择 Cleric，核心不是因为它是个会回答问题的 bot，而是它能嵌进现有工作流：

- 集成 **Kubernetes**
- 集成 **Datadog**
- 集成 **PagerDuty**
- 集成 **Confluence**
- 直接进入 **Slack alert channel** 工作

也就是说，它不是把工程师拉去一个新系统里，而是 **在原来的告警现场直接给调查结果和可执行 insight**。

这一点非常关键。很多所谓“AI 运维”方案最后死在 workflow friction 上：工具能做事，但工程师不用。Cleric 这里显然是反着来，先贴近现有沟通面。

## 部署路径
BlaBlaCar 不是一上来就全量铺开，而是分阶段验证：

### 第一阶段：受控实验
- **2024 年 8 月**，Cleric 先进入 SRE helpdesk 和 alerting channels
- 先处理来自 **Chaos app** 的告警
- 这个 Chaos app 是内部 demo 应用，用来模拟 Kubernetes 生产故障场景

### 第二阶段：进入真实团队
- **2024 年 10 月底**，进入 **Database Reliability 团队** 的告警频道
- 开始承担 **level one incident response**
- 学会按 DBRE 工程师的方式理解 time series 和日志

### 第三阶段：扩到更多团队
- **2025 年 1 月底**，IAM 团队接入
- 有资深工程师对调查结果做 1-5 分打分
- 三周内，Cleric 就拿到第一次满分，成功诊断 “Upstream Retry Limit Exceeded” 告警
- 到 **2025 年 3 月**，又有两个团队接入

到文中时间点：

- 已覆盖约 **10% 的总告警量**
- 大约 **1400 alerts / 月**
- 页面顶部还给了一个运行数据：**2039 次告警调查、3 个活跃团队、85.6% 高置信发现**

## 效果数据
文章里最值得记的是他们怎么衡量效果。

### 1. time to first value
他们定义了一个很实用的指标：

> 当 Cleric 对某类告警的调查质量，能够稳定达到值班工程师水平时，就是到达 time to first value。

结果：

- 在 **IAM 团队**，达到这个点用了 **6 周**
- 在 **Engage 团队**，只用了 **3 周**

这个指标比“模型准确率”实在得多，因为它直接对应组织是否愿意把一类告警真的交给 AI 先处理。

### 2. IAM 团队样本
在 **2025-02-12 到 2025-04-10** 之间：

- Cleric 为 IAM 团队做了 **553 次调查**
- 收到 **48 份工程师反馈**
- 对以下问题效果最好：
  - deployment failures
  - pod crashes
  - application scaling issues
  - job failures
- 在这些调查里，工程师认为 **78% 至少包含一个可执行 insight**

但它不是全能：

- 对 **SLO burn-rate breaches** 和 **anomaly detection** 这类更高阶、模式更复杂的问题
- 只有 **50%** 的调查能提供有用发现

这个结果挺可信，也挺重要：**AI 在“常见、重复、证据模式清晰”的告警上最先成熟；在抽象层更高、上下文更复杂的 incident 上，价值还没那么稳定。**

## 我觉得最值得记的点

### 1. 这不是“替代 on-call”，而是接一级调查
文章最稳的一点是定位很清楚：

- AI 先做 **level one incident response**
- 把 obvious 的东西立刻找出来
- 缩小搜索空间
- 指出相关数据源
- 真正复杂的问题再交给人

这个边界感很健康。它不会神化 AI，也不会强迫组织一次性改完流程。

### 2. Slack 是关键界面，不只是通知渠道
BlaBlaCar 明确说希望“everything centralized in Slack”。这说明他们想要的不是另一个 dashboard，而是：

- 告警来了
- AI 在现场调查
- 结果直接回到工程师日常沟通流里

对 incident response 来说，这比“另做一个智能平台”现实得多。

### 3. 真正的价值之一，是帮助不那么资深的人
文中一个判断我很认同：

- 对 senior engineer，Cleric 省的是“每个 alert 的几分钟”
- 对 junior engineer 或对基础设施不熟的人，Cleric 缩短的是 **调查起点到有效线索之间的整段距离**

也就是说，AI 的杠杆不只是省时间，更是 **降低排障门槛、拉平基础能力下限**。

### 4. 组织知识沉淀，比单次提效更重要
Maxime 的原话里有个很关键的方向：

- 如果一个团队已经解决过某类告警
- 这些经验应该能迁移到别的团队

这本质上是在把 incident response 从“个人经验”往“组织级 operational memory” 推。这个方向比单纯做一个问答 bot 强很多。

### 5. 适合的场景和不适合的场景，已经开始分层
从数据看，Cleric 当前更擅长：

- deployment failure
- pod crash
- scaling issue
- job failure

也就是：

- 信号清晰
- 排查路径相对标准化
- 日志 / 指标 / K8s 状态之间映射明确

而对 burn rate、异常检测这类问题，现阶段显著更难。这种分层很值得参考：**不要用一个统一 KPI 看待所有 incident 类型。**

## 一句话总结
**BlaBlaCar 的这个案例证明，AI 在事故响应里最先成立的形态，不是“全自动 SRE”，而是嵌进 Slack、连接 observability 栈、承担一级调查、把资深工程师经验放大的 incident companion。**

它最先吃掉的是重复调查和信息收敛，而不是最高难度的根因推理。这个落点很现实，也因此更可信。
