# Claude Agent SDK：SRE 事故响应 Agent

- **来源:** https://platform.claude.com/cookbook/claude-agent-sdk-03-the-site-reliability-agent
- **作者:** Ben Lehrburger, Isabella He
- **日期:** 2026-02-16
- **出处:** Anthropic Claude Cookbook
---

## 概览
这篇 cookbook 讲的是一个很实用的方向：怎么把 Claude Agent SDK 从“会看监控、会读日志”的观察型 agent，推进到“能查问题、能改配置、能重启服务、还能写 postmortem”的 SRE 事故响应 agent。

核心不是写一个很复杂的 prompt，而是搭一套受控的执行环境：用 MCP server 暴露有限工具，用 hook 做写操作校验，用 Docker + Prometheus 构造可复现的事故现场，再让 agent 在这个环境里完成调查、修复和记录。

一句话总结：**agent 的价值不在于会不会说，而在于它能不能在安全边界内完成完整闭环。**

## 核心设计

### 1. 从只读 observability agent，升级到可执行 remediation agent
Notebook 02 里 agent 主要负责读外部系统；这篇进一步开放了写能力，让 agent 可以：
- 查询 Prometheus 指标
- 查看容器和应用日志
- 读取配置文件
- 修改配置
- 执行受限 shell 命令
- 输出 postmortem

这里最重要的变化不是“多了几个工具”，而是 **agent 开始能改变系统状态**。一旦能写，安全模型就必须跟着升级。

### 2. Claude Agent SDK + MCP 是整个系统骨架
整套架构很清晰：
- Claude Agent SDK 负责 agent loop
- MCP server 作为子进程提供工具
- 工具通过 JSON-RPC over stdin/stdout 调用
- agent 根据 tool description 自主决定调用顺序

文章反复强调一个点：**真正驱动 agent 行为的，往往是工具定义和描述，而不是长篇 prompt。**

### 3. 系统 prompt 可以很简单，重点放在工具设计
这个 demo 的 prompt 只给了一个通用调查思路：先做健康检查，再看指标，再查日志，再关联根因。并没有写死“必须先调用哪个 tool，再查哪个 query”。

作者的结论很明确：
- prompt 提供方向
- tool descriptions 提供可执行能力
- agent loop 负责多步推理和自适应决策

也就是说，**不要把流程细节全塞进 prompt，应该把能力和约束沉到工具层。**

## 环境与演示场景

### 1. 用本地 Docker 环境模拟真实事故
文章没有直接拿线上系统开刀，而是用本地基础设施做了一个可复现的 SRE 训练场：
- PostgreSQL
- FastAPI API server
- traffic generator
- Prometheus

同时生成这些关键文件：
- `config/docker-compose.yml`
- `config/prometheus.yml`
- `config/api-server.env`
- `services/api_server.py`
- `scripts/traffic_generator.py`
- `hooks/`

这个设计很对：**先让 agent 在可控环境里跑通调查-修复闭环，再考虑接生产系统。**

### 2. 模拟的事故是数据库连接池耗尽
他们故意把 `DB_POOL_SIZE` 从 20 改成 1，制造一个典型线上事故。结果会出现：
- 500 错误率飙升
- P99 延迟显著上升
- DB connection pool 耗尽
- 容器日志出现 timeout / connection error

这是个很好的示例，因为它要求 agent 不能只看单一信号，而要同时关联：
- metrics
- logs
- config
- deployment state

这正好体现 agent 的强项：**跨信号源综合诊断。**

## MCP 工具设计

### 1. 工具按职责分组，而不是胡乱暴露底层接口
文中 MCP server 注册了 12 个核心工具，分成四类：

- **Prometheus**
  - `query_metrics`
  - `list_metrics`
  - `get_service_health`
- **Infrastructure**
  - `read_config_file`
  - `edit_config_file`
  - `run_shell_command`
  - `get_container_logs`
- **Diagnostics**
  - `get_logs`
  - `get_alerts`
  - `get_recent_deployments`
  - `execute_runbook`
- **Documentation**
  - `write_postmortem`

这个分组说明他们不是简单把 API 原样暴露，而是在按 SRE 的任务流设计工具面。

### 2. 关键不是工具数量，而是 tool description 质量
作者明确说：每个工具的 JSON Schema 和 description 都很关键，因为 agent 靠这个判断：
- 什么时候用
- 参数怎么填
- 返回值代表什么

这里的经验很值得记：**清楚的工具描述，比 elaborate prompt 更能提高 agent 自主性。**

### 3. MCP subprocess 隔离很有价值
MCP server 单独跑成子进程，和 agent loop 分离。这样做的好处：
- handler 崩了不会直接拖死 agent
- 工具层和推理层职责清楚
- 更容易替换底层系统接入

这是很典型的 production-minded 设计，不是 notebook 玩具。

## 安全模型：这篇最值得看的部分

### 1. 写能力不是直接放开，而是层层收口
这篇最核心的工程价值在于：它不是粗暴给 agent shell 权限，而是做了多层安全控制。

第一层是 tool handler 限制：
- `edit_config_file` 只能写 `config/` 目录
- `run_shell_command` 只允许 `docker-compose` / `docker` 前缀
- `get_container_logs` 对 container name 做白名单校验

这说明一个原则：**给 agent 的不是“机器权限”，而是“任务权限”。**

### 2. Hook 是第二道防线，校验内容而不是只校验路径
除了 handler 限制，文章还加了 `PreToolUse` hooks，在工具真正执行前再做一次内容级验证。

示例里有两个 hook：
- `validate_pool_size.sh`：拦截危险的 `DB_POOL_SIZE` 修改
- `validate_config_before_deploy.sh`：在 redeploy 前检查配置是否合法

这点非常重要。很多系统只做“能改哪里”的限制，但这里进一步做了“改成什么也要合法”的校验。

也就是：
- handler 控制范围
- hook 控制内容

这套组合比单纯 allowlist 要靠谱得多。

### 3. 调查和修复拆开，是很合理的人类在环设计
他们没有让 agent 一上来就边查边改，而是先让 agent 做 investigation-only，再进入 remediation step。

我觉得这是非常对的默认策略：
- **调查阶段** agent 可以更自主
- **修复阶段** 应该保留人工确认点

因为诊断错了，最多浪费一些 token；修错了，可能直接把事故放大。

## Agent 是怎么工作的

### 1. Baseline：先在健康系统上跑一遍
正式制造事故前，先让 agent 对健康环境做一次检查，确认：
- 能正常连上工具
- metrics 在正常范围
- 服务整体健康

这一步其实相当于给 agent 建 baseline，也是给人建信心。

### 2. Investigation：agent 自主决定调查路径
当事故触发后，文章给 agent 的提示很模糊，类似“api-server 看起来有问题，你去看看”。

然后 agent 自己完成：
- 健康检查
- 指标查询
- 错误率分析
- DB 连接池状态判断
- 日志确认
- 配置文件根因定位

文章特别强调一个观察：**调查路径不是预编排的，而是 agent 根据上一步结果动态决定下一步。**

这正是 agent loop 的意义：
`observe → reason → act → repeat`

### 3. Remediation：修配置、重部署、回查指标、写 postmortem
在 remediation 阶段，agent 应该完成四步：
1. 把 `DB_POOL_SIZE` 改回安全值
2. redeploy api-server
3. 重新检查健康指标，确认恢复
4. 生成事故 postmortem

这个闭环特别关键，因为它不是“找到了根因就结束”，而是 **修复 + 验证 + 文档化** 一起做完。

## 文章想传达的几个核心观点

### 1. Tool descriptions 比复杂 prompt 更重要
这几乎是全文最明确的观点之一。agent 能自己构造 PromQL、自己判断什么时候查日志，不是因为 prompt 里写得细，而是因为工具定义足够清楚。

这对很多 agent 系统都是提醒：**别一味卷 prompt，先把工具接口和描述写明白。**

### 2. Agent 的强项是跨多源信号做综合判断
单看 metrics，可能只能知道“有问题”；
单看 logs，可能只能看到 timeout；
单看 config，可能一眼看不出哪里出错。

但 agent 可以把这些东西串起来，最终定位到 `DB_POOL_SIZE=1` 才是根因。这个能力在运维、排障、审计场景里很有价值。

### 3. 真正可落地的 write access，必须是 scoped write access
作者强调的不是“让 agent 能写”，而是 **让 agent 在受限边界里写**。这包括：
- 限定目录
- 限定命令前缀
- 参数 schema
- 内容 hook 校验
- 分阶段授权

这也是从 demo 走向 production 的关键。

### 4. 技能（skills）和 runbook 是把团队经验喂给 agent 的正确方式
后半部分提到一个很实用的扩展方向：把 runbook、postmortem 模板、升级策略做成 markdown skills，让 agent 自动遵循组织里的既有流程。

这个思路很对，因为通用 agent 只会“会做事”，但不一定“按你团队的方法做事”。skills 就是在补这一层组织经验。

## 生产化扩展思路

### 1. PagerDuty、Confluence、Slack 都可以作为 MCP 扩展
文章给了一个很清楚的 production shape：
- PagerDuty 负责 incident lifecycle
- Confluence 负责 postmortem / knowledge base
- Slack 负责和工程师交互

而且这些工具是按环境变量条件注册的，凭证配置好就自动出现在 `tools/list` 里。

这个模式很优雅：**agent 核心不变，只扩展工具面。**

### 2. 同一个模式可以迁移到很多运维/业务流程
作者最后给了一堆扩展方向，我觉得可以抽象成一句话：

只要任务满足“有多源信号 + 有明确动作 + 能做边界约束”，都适合用这套模式。

比如：
- security incident triage
- deployment verification
- compliance auditing
- cost optimization
- email / support / invoice 这类半结构化流程

## 我认为最值得抄的点

### 1. 先做可复现的本地事故训练场
不要直接拿生产系统给 agent 练手。先用 Docker + metrics + logs 跑通最小闭环。

### 2. 把安全做在工具层，而不是只靠 prompt 说“请谨慎”
真正有效的是目录限制、命令 allowlist、hook 校验，不是道德劝告。

### 3. 调查和修复分阶段
investigation 可以放权，remediation 要保留确认点。这是非常实用的默认策略。

### 4. 用 skills 承载组织知识
工具解决“能做什么”，skills 解决“应该怎么做”。

## 一句话总结
这篇文章最有价值的地方，不是展示“Claude 会修线上事故”，而是给出了一套相对靠谱的工程方法：**用 Claude Agent SDK + MCP + scoped write tools + hooks + skills，把 agent 从观察者变成受控执行者。** 这才是 SRE/运维类 agent 真正能落地的样子。
