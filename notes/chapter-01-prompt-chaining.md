# Chapter 1: Prompt Chaining

> Book: Agentic Design Patterns  
> Topic: Prompt Chaining / Pipeline Pattern  
> Goal: 用新手友好的方式理解 Prompt Chaining，以及它为什么是构建 AI Agent 的基础模式。

---

## Table of Contents

- [1. One-Sentence Summary](#1-one-sentence-summary)
- [2. What Is Prompt Chaining?](#2-what-is-prompt-chaining)
- [3. A Milk Tea Shop Analogy](#3-a-milk-tea-shop-analogy)
- [4. Why Single Prompts Break Down](#4-why-single-prompts-break-down)
- [5. How Sequential Decomposition Improves Reliability](#5-how-sequential-decomposition-improves-reliability)
- [6. Why Structured Output Matters](#6-why-structured-output-matters)
- [7. Practical Applications](#7-practical-applications)
- [8. Hands-On Code Example: What It Teaches](#8-hands-on-code-example-what-it-teaches)
- [9. Prompt Chaining and Context Engineering](#9-prompt-chaining-and-context-engineering)
- [10. When To Use This Pattern](#10-when-to-use-this-pattern)
- [11. Common Beginner Mistakes](#11-common-beginner-mistakes)
- [12. Reusable Prompt Templates](#12-reusable-prompt-templates)
- [13. Key Takeaways](#13-key-takeaways)
- [14. References](#14-references)

---

## 1. One-Sentence Summary

Prompt Chaining is the practice of breaking a complex LLM task into a sequence of smaller, focused steps, where each step produces an intermediate output that becomes the input for the next step.

用中文说：

> Prompt Chaining 的本质不是“多写几个 Prompt”，而是把复杂任务设计成一条可靠、可检查、可组合的工作流。

---

## 2. What Is Prompt Chaining?

Prompt Chaining，也叫 Pipeline Pattern，是一种处理复杂 LLM 任务的基础模式。

它不要求模型用一个超长 Prompt 一次性完成所有事情，而是把任务拆成多个更小、更清晰的子任务。

典型结构如下：

```text
Prompt 1 -> Intermediate Output 1
Prompt 2 -> Intermediate Output 2
Prompt 3 -> Final Output
```

例如，一个复杂任务是：

```text
分析市场研究报告，并总结趋势，再写一封邮件给营销团队。
```

如果直接交给一个 Prompt，模型需要同时做：

```text
理解报告
总结发现
识别趋势
提取数据点
组织邮件语言
控制语气和格式
```

Prompt Chaining 会把它拆成：

```text
Step 1: 总结报告关键发现
Step 2: 从总结中识别趋势和支持数据
Step 3: 基于趋势和数据写邮件
```

每一步只处理一个子问题。这样模型更容易完成任务，系统也更容易检查哪一步出了问题。

---

## 3. A Milk Tea Shop Analogy

可以用奶茶店来理解 Prompt Chaining。

假设顾客说：

> 我要一杯少糖、去冰、加珍珠、加奶盖、燕麦奶、打包带走的奶茶。

如果让一个店员凭记忆从头到尾完成，可能会漏掉一些要求，比如忘记燕麦奶，或者忘记去冰。

更稳定的方式是把流程拆开：

```text
点单员：确认需求
    ↓
制茶员：制作茶底
    ↓
配料员：加入珍珠和奶盖
    ↓
质检员：检查少糖、去冰、燕麦奶
    ↓
打包员：封杯交付
```

这就是 Prompt Chaining 的直觉。

关键不是“步骤变多”，而是：

- 每一步职责更清楚。
- 每一步输入输出更明确。
- 出错时更容易定位。
- 中间结果可以被检查和修正。
- 整个流程更稳定。

---

## 4. Why Single Prompts Break Down

原文指出，对于多步骤、多约束的任务，单个复杂 Prompt 容易导致以下问题。

| Problem | 中文解释 | 例子 |
| --- | --- | --- |
| Instruction Neglect | 忽略指令 | 要求总结、提取数据、写邮件，但模型只完成了总结 |
| Contextual Drift | 上下文漂移 | 一开始分析市场报告，后面变成泛泛而谈营销建议 |
| Error Propagation | 错误传播 | 早期理解错数据，后续趋势和邮件都被带偏 |
| Long Context Pressure | 上下文窗口压力 | 报告太长、要求太多，模型难以稳定处理所有信息 |
| Hallucination | 幻觉增加 | 找不到数据时，模型编造看似合理的数据点 |

### 4.1 Instruction Neglect

当一个 Prompt 里塞进太多要求时，模型可能会漏掉其中一部分。

例如你要求模型：

```text
分析报告
总结关键发现
识别趋势
提取数据点
写一封邮件
语气要正式
格式要简洁
```

模型可能总结得不错，但忘记提取数据点，或者邮件格式不符合要求。

### 4.2 Contextual Drift

复杂 Prompt 做到后面，模型可能逐渐偏离最初目标。

例如一开始要求分析市场研究报告，最后输出却变成一般性的营销建议。

更严谨地说，这不只是 attention “天然偏向开头”的问题。实际原因通常包括：

- 上下文太长导致信息竞争。
- 任务目标太多导致优先级不清。
- 关键信息被埋在上下文中间。
- 模型在生成过程中逐渐偏离初始约束。

Prompt Chaining 的作用是把每一步的上下文变短、目标变清楚，从而降低漂移概率。

### 4.3 Error Propagation

如果模型在早期理解错了，后续内容可能都会沿着错误继续生成。

例如模型一开始误读了市场报告里的关键数据，那么后面的趋势分析和邮件都会建立在错误基础上。

需要注意：Prompt Chaining 本身也可能传播错误。如果 Step 1 的输出错了，Step 2 仍然可能继续放大这个错误。

因此一个可靠的 chain 通常需要：

- 中间结果验证
- schema 校验
- 工具校验
- 条件重试
- 必要时人工审核

---

## 5. How Sequential Decomposition Improves Reliability

原文用市场研究报告作为例子。

原始任务：

```text
分析一份市场研究报告，总结关键发现，识别趋势，提取支持数据点，并起草一封发给营销团队的邮件。
```

不要一次性让模型完成全部工作，而是拆成三个阶段。

### Step 1: Summarization

```text
Summarize the key findings of the following market research report:
[text]
```

这一阶段只负责总结报告，不负责写邮件，也不负责包装语言。

### Step 2: Trend Identification

```text
Using the summary, identify the top three emerging trends and extract the specific data points that support each trend:
[output from step 1]
```

这一阶段基于 Step 1 的总结继续分析，目标更窄。

### Step 3: Email Composition

```text
Draft a concise email to the marketing team that outlines the following trends and their supporting data:
[output from step 2]
```

这一阶段只负责表达和组织。

### Why This Works

这种拆解更可靠，因为：

- 每个 Prompt 的目标更单一。
- 每一步的输出更容易检查。
- 错误更容易定位到具体步骤。
- 可以在步骤之间插入验证逻辑。
- 可以为不同步骤分配不同角色。

例如：

| Step | Role | Responsibility |
| --- | --- | --- |
| Step 1 | Market Analyst | 总结市场报告 |
| Step 2 | Trend Analyst | 识别趋势和支持数据 |
| Step 3 | Expert Documentation Writer | 写出清晰专业的邮件 |

原文中使用了 `Trade Analyst`，但结合上下文，这里更合理地理解为 `Trend Analyst`，也就是趋势分析师。

---

## 6. Why Structured Output Matters

Prompt Chain 的可靠性，很大程度取决于步骤之间传递的数据是否清晰、稳定、可解析。

如果上一步输出的是模糊自然语言，下一步就可能误读。

例如：

```text
消费者越来越喜欢个性化服务，也更关注环保品牌，很多数据都支持这个变化。
```

这段话人能看懂，但对下一步处理不够稳定，因为它没有清楚说明：

- 趋势名称是什么。
- 支持数据是什么。
- 哪个数据支持哪个趋势。
- 是否还有其他趋势被遗漏。

更好的方式是使用 JSON 或 XML 这样的结构化格式。

```json
{
  "trends": [
    {
      "trend_name": "AI-Powered Personalization",
      "supporting_data": "73% of consumers prefer to do business with brands that use personal information to make their shopping experiences more relevant."
    },
    {
      "trend_name": "Sustainable and Ethical Brands",
      "supporting_data": "Sales of products with ESG-related claims grew 28% over the last five years, compared to 20% for products without."
    }
  ]
}
```

结构化输出的好处：

- 字段含义明确。
- 数据关系清楚。
- 后续步骤更容易解析。
- 更容易做 schema 校验。
- 更适合接入程序、数据库和工具。

一句话总结：

> 在 Prompt Chaining 里，中间结果不是散文，而是传给下一个环节的标准数据。

---

## 7. Practical Applications

原文列出了 7 类典型应用场景。

### 7.1 Information Processing Workflows

信息处理任务通常不是问一句答一句，而是要对原始信息进行多次加工。

典型流程：

```text
从 URL 或文档提取文本
    ↓
清洗文本
    ↓
总结重点
    ↓
提取实体
    ↓
查询内部知识库
    ↓
生成最终报告
```

适用场景：

- 自动内容分析
- AI Research Assistant
- 长文档总结
- 复杂报告生成

### 7.2 Complex Query Answering

有些问题不能靠单个事实回答，需要多步推理或多来源信息整合。

原文例子：

```text
What were the main causes of the stock market crash in 1929,
and how did government policy respond?
```

这个问题至少包含两个子问题：

```text
1929 年股市崩盘的主要原因是什么？
政府政策如何回应？
```

可以拆成：

```text
识别用户问题中的子问题
    ↓
检索或研究股市崩盘原因
    ↓
检索或研究政府政策回应
    ↓
综合成完整答案
```

这里还涉及一个重要点：复杂系统经常混合使用并行和串行。

可以并行的部分：

```text
文章 A -> 提取信息
文章 B -> 提取信息
文章 C -> 提取信息
```

必须串行的部分：

```text
汇总提取结果
    ↓
综合成草稿
    ↓
审查和修改
    ↓
生成最终报告
```

所以 Prompt Chaining 不等于所有步骤都排队执行。更准确地说：

> 独立的信息收集可以并行，有依赖关系的综合、推理、修改适合串行。

### 7.3 Data Extraction and Transformation

这类任务的目标是把非结构化文本转换成结构化数据。

例如从发票、邮件、PDF 表单中提取：

- name
- address
- amount
- date
- order id

典型流程：

```text
Prompt 1: 尝试提取字段
    ↓
Processing: 检查字段是否完整、格式是否正确
    ↓
Prompt 2: 如果缺失或格式错误，专门补充提取问题字段
    ↓
Processing: 再次验证
    ↓
Output: 输出经过验证的结构化数据
```

这一节的关键是：中间可以插入确定性处理逻辑。

例如：

- 字段是否为空
- 日期格式是否正确
- 金额是否是数字
- 是否满足 JSON schema
- 是否需要调用计算器

LLM 适合理解文本、提取字段、规范化表达；工具更适合精确计算、格式校验、数据库验证。

### 7.4 Content Generation Workflows

复杂内容生成天然适合拆成多个阶段。

正常写作也不是一步完成，而是：

```text
想选题
    ↓
列大纲
    ↓
写初稿
    ↓
分段扩展
    ↓
整体修改
    ↓
润色风格
```

原文流程：

```text
Prompt 1: 根据用户兴趣生成 5 个主题
Processing: 用户选择一个，或系统自动选择最佳主题
Prompt 2: 基于选定主题生成详细大纲
Prompt 3: 根据大纲第一点写第一部分草稿
Prompt 4: 根据大纲第二点写第二部分草稿，并提供前文作为上下文
Prompt 5: 审查并优化完整草稿
```

适用内容：

- 创意叙事
- 技术文档
- 视频脚本
- 长篇文章
- 结构化报告

### 7.5 Conversational Agents with State

有状态对话 Agent 不能只看用户最后一句话，而是要维护 conversation state。

例如用户说：

```text
我想订一杯奶茶。
```

系统识别：

```json
{
  "intent": "order_drink",
  "drink": "milk_tea"
}
```

然后系统发现信息不完整，继续问：

```text
请问甜度和冰量怎么选？
```

用户回答：

```text
少糖，去冰。
```

系统更新状态：

```json
{
  "intent": "order_drink",
  "drink": "milk_tea",
  "sugar": "less_sugar",
  "ice": "no_ice"
}
```

这个流程本质上也是 Prompt Chaining：

```text
识别用户意图和实体
    ↓
更新 state
    ↓
根据当前 state 回复或追问缺失信息
    ↓
下一轮继续更新 state
```

适用场景：

- 客服
- 预约系统
- 下单系统
- 表单填写
- 多轮任务执行

### 7.6 Code Generation and Refinement

代码生成也适合拆成多步流程。

```text
理解需求
    ↓
生成伪代码或设计大纲
    ↓
写初版代码
    ↓
检查潜在错误
    ↓
重写或优化
    ↓
添加文档或测试
```

这类流程的优势是可以在 LLM 调用之间插入工程工具：

- formatter
- linter
- type checker
- unit tests
- static analysis
- code review step

这说明 Prompt Chaining 不只是 Prompt 技巧，也是一种系统编排方式。

### 7.7 Multimodal and Multi-Step Reasoning

多模态任务可能同时包含：

- 图片
- 图中文字
- 标签
- 表格
- 结构化数据

原文例子是：图片中有嵌入文字，标签标注了某些文本片段，旁边还有表格解释每个标签的含义。

可以拆成：

```text
Prompt 1: 提取并理解图片中的文字
Prompt 2: 把图片文字和对应标签连接起来
Prompt 3: 结合表格解释信息，得出最终输出
```

多模态任务难点不只是“图片难识别”，更难的是：

- 不同信息源之间有对应关系。
- 这些关系需要逐步建立。
- 最终答案依赖前面每一步是否正确。

---

## 8. Hands-On Code Example: What It Teaches

原文后半部分给了一个 LangChain / LangGraph 示例。

示例任务：

```text
输入一段笔记本电脑规格描述
    ↓
Prompt 1: 提取技术规格
    ↓
Prompt 2: 把规格转换成 JSON
```

核心代码结构可以理解为：

```python
extraction_chain = prompt_extract | llm | StrOutputParser()

full_chain = (
    {"specifications": extraction_chain}
    | prompt_transform
    | llm
    | StrOutputParser()
)
```

这段代码的重点不是语法，而是说明：

- 一个 chain 可以作为另一个 chain 的输入。
- 中间结果可以由框架接住并继续传递。
- Prompt Chaining 可以从手工流程变成可执行工作流。
- LangChain 适合线性链条，LangGraph 更适合有状态或循环的 Agent 工作流。

---

## 9. Prompt Chaining and Context Engineering

原文还引入了 Context Engineering。

Prompt Engineering 更关注：

```text
如何把当前这句话问好？
```

Context Engineering 更关注：

```text
在模型生成之前，如何构建完整、准确、可用的信息环境？
```

Context 可以包括：

- system prompt
- retrieved documents
- tool outputs
- user identity
- interaction history
- environmental state

Prompt Chaining 和 Context Engineering 的关系很密切。

每一步 Prompt Chain 都在做一件事：

```text
生成、筛选、压缩、转换或验证下一步需要的上下文。
```

因此，Prompt Chaining 可以理解为 Context Engineering 的一种执行方式。

它不只是把 Prompt 串起来，而是在逐步构造一个更适合模型完成任务的信息环境。

---

## 10. When To Use This Pattern

适合使用 Prompt Chaining 的情况：

- 一个任务包含多个明显阶段。
- 单个 Prompt 太长、太复杂。
- 中间结果需要检查或验证。
- 需要外部工具、API、数据库或搜索系统。
- 需要结构化输出。
- 需要多步推理、规划或状态管理。
- 需要把独立任务和依赖任务组合起来。

不一定适合的情况：

- 任务很简单，一步就能稳定完成。
- 拆分后只增加延迟，没有提升质量。
- 每一步没有清晰输入输出。
- 没有验证机制，错误会沿着链条继续传播。

Rule of thumb:

> 如果一个 Prompt 里同时出现多个不同目标，先考虑拆链，而不是继续堆指令。

---

## 11. Common Beginner Mistakes

### Mistake 1: 把 Prompt Chaining 理解成“多问几次”

Prompt Chaining 不是随便多问几轮，而是有明确输入输出契约的工作流。

### Mistake 2: 每一步都交给 LLM

链条里不一定每一步都是 LLM。

可以插入：

- 普通代码
- API
- 数据库查询
- 搜索系统
- schema validator
- calculator
- human review

### Mistake 3: 只拆步骤，不设计数据格式

如果中间输出不稳定，下一步仍然会失败。

结构化输出是 Prompt Chaining 的关键配套能力。

### Mistake 4: 以为 Chaining 会自动消除错误

Prompt Chaining 只是让错误更容易定位和控制，不会自动保证正确。

可靠的 chain 需要验证、重试、工具调用和必要的人类监督。

### Mistake 5: 过度拆分

不是步骤越多越好。

如果拆分后的每一步都没有独立价值，反而会让系统变慢、变复杂、维护成本更高。

---

## 12. Reusable Prompt Templates

下面是一个市场研究报告任务的三步 Prompt Chain 模板。

### 12.1 Step 1: Summarization

```text
You are a market analyst.

Task:
Summarize the key findings of the following report.

Constraints:
- Focus only on factual findings.
- Do not write recommendations yet.
- Do not draft an email.
- Output 5-8 bullet points.

Input:
[report text]
```

### 12.2 Step 2: Trend Identification

```text
You are a trend analyst.

Task:
Based on the summary below, identify the top 3 emerging trends.

Output format:
Return valid JSON with this schema:
{
  "trends": [
    {
      "trend_name": "...",
      "explanation": "...",
      "supporting_data": "..."
    }
  ]
}

Constraints:
- Use only information from the provided summary.
- Do not invent data.
- Each trend must include supporting data.

Input:
[summary from step 1]
```

### 12.3 Step 3: Email Composition

```text
You are an expert documentation writer.

Task:
Draft a concise email to the marketing team based on the trends and supporting data below.

Constraints:
- Keep the tone professional.
- Keep it under 180 words.
- Mention only the trends and data provided.
- Do not add unsupported claims.

Input:
[JSON from step 2]
```

---

## 13. Key Takeaways

- Prompt Chaining breaks complex tasks into smaller, focused steps.
- Each step can be an LLM call, deterministic processing logic, external tool call, or human review.
- The output of one step becomes the input of the next step.
- This pattern improves reliability, interpretability, and debuggability.
- Structured output is critical because it reduces ambiguity between steps.
- Prompt Chaining is foundational for building multi-step Agentic systems.
- It works best when paired with validation, tool use, state management, and context engineering.

Final summary:

> Prompt Chaining turns LLM work from a single-shot answer into a manageable workflow. Its real value is not adding more prompts, but making each step clear, verifiable, and composable.

---

## 14. References

- LangChain LCEL Documentation: https://python.langchain.com/v0.2/docs/core_modules/expression_language/
- LangGraph Documentation: https://langchain-ai.github.io/langgraph/
- Prompt Engineering Guide - Chaining Prompts: https://www.promptingguide.ai/techniques/chaining
- OpenAI API Documentation: https://platform.openai.com/docs/guides/gpt/prompting
- CrewAI Documentation: https://docs.crewai.com/
- Google AI for Developers: https://cloud.google.com/discover/what-is-prompt-engineering?hl=en
- Vertex Prompt Optimizer: https://cloud.google.com/vertex-ai/generative-ai/docs/learn/prompts/prompt-optimizer

