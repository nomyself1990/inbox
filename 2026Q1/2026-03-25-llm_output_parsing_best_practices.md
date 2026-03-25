# LLM 输出解析与结构化处理

## 核心问题

LLM 的输出本质上是非确定性的文本流。当你需要从中提取结构化数据时，面临两类风险：结构不合法（JSON 语法错误、缺字段）和语义不正确（结构合法但内容不对）。

## 我的技术选择

### 分层策略

三层防御，逐层兜底：

```
第一层：Structured Output / Tool Use（模型层，保证结构合法）
第二层：Zod Schema 验证（应用层，校验语义约束）
第三层：LLM 重试（兜底，把验证错误反馈给模型修正）
```

第一层和第二层覆盖绝大多数情况，第三层极少触发。如果重试频繁说明 schema 设计或 prompt 有问题，应该从源头修。

### Structured Output vs Tool Use：如何选择

两者组合使用，按场景区分：

**Structured Output** 用于纯数据提取场景。模型的唯一任务就是按 schema 输出结构化数据，没有"决策"成分。典型场景：文本分类、实体提取、内容摘要。

**Tool Use** 用于模型需要同时决定"做什么"和"输出什么"的场景。模型根据上下文选择调用哪个 tool，tool 的参数 schema 约束输出结构。典型场景：agent 工作流、多步骤任务编排。

Anthropic 目前的 structured output 实际上是通过 tool use 机制实现的，两者在底层没有本质区别。OpenAI 则提供了独立的 `response_format` 路径。实践中对于纯提取任务，OpenAI 用 `response_format`，Anthropic 用单 tool 强制调用，效果等价。

### Schema 定义：Zod

所有项目统一用 Zod 定义 schema，一处定义同时获得运行时验证和 TypeScript 类型推导。通过 `zod-to-json-schema` 转换后传给模型 API。

```typescript
import { z } from "zod"

const AnalysisSchema = z.object({
  sentiment: z.enum(["positive", "negative", "neutral"]),
  confidence: z.number().min(0).max(1),
  keywords: z.array(z.string()).min(1),
  summary: z.string().max(500),
})

type Analysis = z.infer<typeof AnalysisSchema>
```

Schema 设计原则：字段名语义清晰，枚举值和自然语言对齐，避免过度嵌套。这些直接影响模型的首次成功率。

### SDK 选择：按项目复杂度分

**直接调 Anthropic/OpenAI SDK**：轻量项目、对模型调用有精细控制需求的场景。自己组装 structured output 参数，Zod 验证放在调用后。好处是没有额外抽象层，调试直观。

```typescript
// OpenAI 直接调用
const response = await openai.chat.completions.create({
  model: "gpt-4o",
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "analysis",
      strict: true,
      schema: zodToJsonSchema(AnalysisSchema),
    },
  },
  messages,
})
const parsed = AnalysisSchema.parse(JSON.parse(response.choices[0].message.content))

// Anthropic 通过 tool use
const response = await anthropic.messages.create({
  model: "claude-sonnet-4-20250514",
  tools: [{
    name: "analyze",
    description: "输出分析结果",
    input_schema: zodToJsonSchema(AnalysisSchema),
  }],
  tool_choice: { type: "tool", name: "analyze" },
  messages,
})
```

**走框架（如 Vercel AI SDK）**：需要多模型支持、流式 structured output、或内置重试逻辑的场景。框架统一了不同 provider 的 structured output 接口，重试循环也内置处理。

```typescript
import { generateObject } from "ai"
import { anthropic } from "@ai-sdk/anthropic"
import { openai } from "@ai-sdk/openai"

// 同一套代码，切换 provider 只换一行
const { object } = await generateObject({
  model: anthropic("claude-sonnet-4-20250514"), // 或 openai("gpt-4o")
  schema: AnalysisSchema,
  prompt: "分析以下文本...",
})
// object 已经是类型安全的 Analysis，验证和重试由框架处理
```

### 重试策略

重试逻辑统一在框架层处理，不在业务代码中手写。Vercel AI SDK 的 `generateObject` 在 Zod 验证失败时会自动将错误信息拼回 prompt 让模型修正。

控制参数：最多重试 2-3 次。超过说明问题在 schema 或 prompt 层面，继续重试只是浪费 token。

### 多模型兼容

所有项目默认同时支持 Claude 和 OpenAI 系列模型。实现方式：

直接调 SDK 时，封装一层 provider adapter，统一输入输出接口，内部分别调 Anthropic SDK 和 OpenAI SDK。structured output 的传参方式按 provider 适配。

走框架时，provider 切换是框架内置能力，只需更换 model 参数。

## 各方式详解

### Structured Output

在 API 层传入 JSON Schema，模型的 token 生成过程被 constrained decoding 强制限定在合法范围内。可靠性最高，所有需要结构化输出的场景优先考虑。

局限：仅保证结构合法，不保证语义正确。

### Tool Use / Function Calling

模型对 tool calling 格式做过专门训练，输出质量通常优于 raw JSON。在 agent 场景下是自然选择，因为模型本来就需要决定调用哪个工具。

### Prompt Engineering

在 prompt 中明确描述输出格式，提供 few-shot 示例。单独使用可靠性最低，但和 structured output 组合能提升首次成功率。关键是给完整的输出示例，明确标注必填字段和值域范围，说明边界情况的处理方式。

### 后处理 / 宽容解析

对模型输出做容错修复：去掉 markdown code fence、修复 trailing comma、截断多余文本。适合不支持 structured output 的模型，或流式场景下的中间状态解析。在有 structured output 的前提下很少用到。

## 注意事项

1. Structured output 保证的是 JSON 结构合法，不保证内容正确。语义正确性仍然取决于模型能力和 prompt 质量
2. 重试频繁触发时应该回头检查 schema 复杂度和 prompt 指引，而非增加重试次数
3. 流式场景下 structured output 需要 partial parsing，Vercel AI SDK 的 `streamObject` 已经处理了这个问题
4. Schema 越简单、字段命名越贴近自然语言，模型首次成功率越高
