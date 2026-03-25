# Uber 如何将 AI 融入开发流程

- **来源:** https://newsletter.pragmaticengineer.com/p/how-uber-uses-ai-for-development
- **作者:** Gergely Orosz
- **日期:** 2026-03-10
- **出处:** The Pragmatic Engineer
---

## 概览
Uber 正在系统性地将 AI 集成到整个工程组织中，核心策略是用 AI 自动化繁琐工作，让工程师专注于高价值的创造性任务。

## 关键数据（2026 年 3 月）
- **84%** 的开发者使用 agentic 编码工具
- **65-72%** 的代码通过 IDE 内置工具生成
- Claude Code 使用率从 12 月的 32% 增长到 2 月的 **63%**，近乎翻倍
- **92%** 的开发者每月使用 agent
- **11%** 的 PR 由 AI agent 发起
- AI 成本自 2024 年以来增长了 **6 倍**

## 四层 Agentic 架构

### Layer 1: 内部 AI 平台
基于 Michelangelo（Uber 的 ML 平台）构建，提供模型网关和前沿模型访问能力。

### Layer 2: 内部上下文源
源代码仓库、工程文档、Slack 信息、JIRA 工单。这些作为 agent 的"记忆"。

### Layer 3: 行业 Agent
支持 Claude Code、GitHub Copilot、Codex 等工具，确保能使用最新最强的 AI 能力。

### Layer 4: 专用 Agent
后台 agent 平台、测试生成系统、代码审查 agent、迁移管理工具。

## 内部工具与基础设施

### MCP Gateway
Uber 实现了 Model Context Protocol 网关，提供：通过简单配置代理内部端点，统一第一方和第三方 MCP 接口，集中处理授权、遥测、日志等平台关注点，以及用于发现和实验的注册中心和沙箱。

### Uber Agent Builder
无代码平台，允许工程师访问内部数据源、构建多 agent 工作流、使用 Agent Studio 进行可视化调试、通过注册中心发现和管理 agent 版本。

### AIFX CLI
命令行工具，解决部署挑战：在组织内配置 AI agent、发现和配置 MCP 服务器、运行后台 agent 任务、管理 agent 和客户端的更新。

## 开发者工作流的演变
**传统模式：** 规划 → 编写代码（占大部分时间） → 代码审查
**早期 agentic 模式：** 与单个 agent 单线程交互
**当前 Uber 模式：** 工程师同时编排多个并行 agent。一位工程师的描述很生动：等待某个任务运行时，与其喝咖啡刷 Reddit，不如再启动一个后台 agent。

## 新内部工具
**Code Inbox：** 根据相关性智能路由包含 AI 生成代码的 PR。
**uReview：** 专门针对 AI 生成代码生成高质量的 code review 评论。
**Autocover：** 每月为代码库生成 5000+ 单元测试。
**Shepherd：** 通过 agent 协调端到端管理大规模迁移。

## 战略理念
工程总监 Anshu Chada 的说法：把无聊的工作（升级、迁移、bug 修复）推给 AI 后，工程师满意度提升，能专注于创造之前想都想不到的功能。Uber 将此定位为"消除苦力活"，而非自动化一切。

## 落地挑战
即使在 Uber 这样前沿的组织中，AI 落地仍面临阻力：内部采纳速度低于预期，自上而下的强制推行效果差于同事间的口碑传播，token 成本和资源消耗引发日益增长的担忧，需要持续的平台投入来支撑大规模 agent 使用。

## 核心启示
Uber 的案例表明，成功的大规模 AI 集成需要大量基础设施投入、深思熟虑的工具设计和文化层面的对齐，仅有前沿模型的访问权限远远不够。
