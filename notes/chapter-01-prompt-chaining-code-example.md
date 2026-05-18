# 第一章实战部分：Prompt Chaining 代码案例

> 主题：用 LangChain LCEL 实现一个两步 Prompt Chain  
> 核心任务：从非结构化文本中提取电脑配置，并转换成 JSON 格式。

---

## 中文索引

- [1. 这段代码的核心目标](#1-这段代码的核心目标)
- [2. 原文完整代码](#2-原文完整代码)
- [3. 这个案例在做什么](#3-这个案例在做什么)
- [4. 为什么这个案例适合讲 Prompt Chaining](#4-为什么这个案例适合讲-prompt-chaining)
- [5. 整体流程图](#5-整体流程图)
- [6. 工程视角：这段代码可以拆成 5 个模块](#6-工程视角这段代码可以拆成-5-个模块)
- [7. 模块一：导入依赖](#7-模块一导入依赖)
- [8. 模块二：初始化模型](#8-模块二初始化模型)
- [9. 模块三：定义第一个 Prompt：提取信息](#9-模块三定义第一个-prompt提取信息)
- [10. 模块四：定义第二个 Prompt：转换成 JSON](#10-模块四定义第二个-prompt转换成-json)
- [11. 模块五：构建第一条 Chain](#11-模块五构建第一条-chain)
- [12. 模块六：构建完整 Chain](#12-模块六构建完整-chain)
- [13. 模块七：准备输入文本](#13-模块七准备输入文本)
- [14. 模块八：执行 Chain](#14-模块八执行-chain)
- [15. 模块九：打印结果](#15-模块九打印结果)
- [16. 一个容易被新手忽略的问题](#16-一个容易被新手忽略的问题)
- [17. 这段代码的核心骨架](#17-这段代码的核心骨架)
- [18. 关键代码标亮版](#18-关键代码标亮版)
- [19. 这段代码背后的底层逻辑](#19-这段代码背后的底层逻辑)
- [20. 从业务逻辑看这段代码](#20-从业务逻辑看这段代码)
- [21. 从 Prompt Chaining 主题看这段代码](#21-从-prompt-chaining-主题看这段代码)
- [22. 工程视角：这段代码有什么优点](#22-工程视角这段代码有什么优点)
- [23. 工程视角：这段代码有什么不足](#23-工程视角这段代码有什么不足)
- [24. 如果我要把它改成更工程化，会怎么做](#24-如果我要把它改成更工程化会怎么做)
- [25. 新手最应该看懂的 3 行代码](#25-新手最应该看懂的-3-行代码)
- [26. 一句话总结这段代码](#26-一句话总结这段代码)
- [27. 发布版小结](#27-发布版小结)

---

## 1. 这段代码的核心目标

这一节代码实现的是一个两步 Prompt Chain。

它要完成的任务是：

> 从一段非结构化文本中提取电脑配置，然后把这些配置转换成 JSON 格式。

原始输入是一句话：

```text
The new laptop model features a 3.5 GHz octa-core processor, 16GB of RAM, and a 1TB NVMe SSD.
```

最终目标是得到类似这样的结构化结果：

```json
{
  "cpu": "3.5 GHz octa-core processor",
  "memory": "16GB of RAM",
  "storage": "1TB NVMe SSD"
}
```

这正好对应前面理论部分讲的：

> 不要让模型一次性“理解文本 + 提取信息 + 转换格式”，而是把它拆成两个步骤。

---

## 2. 原文完整代码

```python
import os
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser

# For better security, load environment variables from a .env file
# from dotenv import load_dotenv
# load_dotenv()
# Make sure your OPENAI_API_KEY is set in the .env file

# Initialize the Language Model (using ChatOpenAI is recommended)
llm = ChatOpenAI(temperature=0)

# --- Prompt 1: Extract Information ---
prompt_extract = ChatPromptTemplate.from_template(
    "Extract the technical specifications from the following text:\n\n{text_input}"
)

# --- Prompt 2: Transform to JSON ---
prompt_transform = ChatPromptTemplate.from_template(
    "Transform the following specifications into a JSON object with 'cpu', 'memory', and 'storage' as keys:\n\n{specifications}"
)

# --- Build the Chain using LCEL ---
# The StrOutputParser() converts the LLM's message output to a simple string.
extraction_chain = prompt_extract | llm | StrOutputParser()

# The full chain passes the output of the extraction chain into the 'specifications'
# variable for the transformation prompt.
full_chain = (
    {"specifications": extraction_chain}
    | prompt_transform
    | llm
    | StrOutputParser()
)

# --- Run the Chain ---
input_text = "The new laptop model features a 3.5 GHz octa-core processor, 16GB of RAM, and a 1TB NVMe SSD."

# Execute the chain with the input text dictionary.
final_result = full_chain.invoke({"text_input": input_text})

print("\n--- Final JSON Output ---")
print(final_result)
```

---

## 3. 这个案例在做什么

这个案例展示 Prompt Chaining 如何在工程里变成一个可执行的数据处理流水线。

它把一个任务拆成了两步：

```text
原始文本
    ↓
Step 1：提取技术规格
    ↓
Step 2：转换成 JSON
    ↓
最终结构化结果
```

也就是：

```text
text_input
    ↓
specifications
    ↓
json_result
```

这就是第一章实战部分的核心。

---

## 4. 为什么这个案例适合讲 Prompt Chaining

因为它非常简单，但又完整体现了 Prompt Chaining 的三个关键点：

1. 任务拆解
2. 中间结果传递
3. 输出格式转换

如果不用 Prompt Chaining，我们可能会直接写一个大 Prompt：

```text
请从下面文本中提取 CPU、内存、存储信息，并输出 JSON。
```

这样也能做，但问题是：

- 提取和格式化混在一起。
- 模型可能漏字段。
- 模型可能 JSON 格式不稳定。
- 不容易定位错误。

这段代码的设计思路是：

> 先让模型专心提取信息，再让模型专心转换格式。

这就是前面理论中说的 Sequential Decomposition，顺序拆解。

---

## 5. 整体流程图

可以把这段代码理解成下面这个流程：

```text
输入文本 input_text
    ↓
prompt_extract
    ↓
llm
    ↓
StrOutputParser
    ↓
extraction_chain 输出 specifications
    ↓
prompt_transform
    ↓
llm
    ↓
StrOutputParser
    ↓
final_result
```

更简化一点：

```text
原始文本
    ↓
提取规格
    ↓
规格文本
    ↓
转换 JSON
    ↓
最终结果
```

这个流程非常重要。

因为 Prompt Chaining 的重点不在“调用了几次模型”，而在于：

> 每一步都有明确职责，并且上一步的输出会成为下一步的输入。

---

## 6. 工程视角：这段代码可以拆成 5 个模块

从工程角度看，这段代码不是散乱的。

它可以拆成 5 个部分：

1. 导入依赖
2. 初始化模型
3. 定义两个 Prompt
4. 构建 Chain
5. 执行 Chain

对应代码结构是：

```text
import ...
    ↓
llm = ChatOpenAI(...)
    ↓
prompt_extract = ...
prompt_transform = ...
    ↓
extraction_chain = ...
full_chain = ...
    ↓
final_result = full_chain.invoke(...)
```

下面逐块讲。

---

## 7. 模块一：导入依赖

```python
import os
from langchain_openai import ChatOpenAI
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.output_parsers import StrOutputParser
```

这一部分是在准备工具。

核心有三个：

- `ChatOpenAI`：负责调用大模型。
- `ChatPromptTemplate`：负责创建 Prompt 模板。
- `StrOutputParser`：负责把模型输出转成普通字符串。

新手可以把它理解成准备一个小型 AI 加工厂：

- `ChatPromptTemplate`：准备工单模板。
- `ChatOpenAI`：真正干活的 AI 工人。
- `StrOutputParser`：把 AI 返回的消息拆成普通文本。

LangChain 官方文档中，`ChatPromptTemplate` 的作用是为聊天模型创建灵活的 Prompt 模板；`StrOutputParser` 的作用是把模型返回的消息内容提取成普通字符串。

---

## 8. 模块二：初始化模型

```python
llm = ChatOpenAI(temperature=0)
```

这一行创建了一个模型对象。

后面所有的 Prompt 都会交给这个 `llm` 来处理。

### temperature=0 是什么意思

`temperature` 可以简单理解成模型回答的“随机程度”。

- `temperature` 越高：回答越发散、越有创意。
- `temperature` 越低：回答越稳定、越确定。

这里设置为：

```python
temperature=0
```

意思是希望模型尽量稳定。

因为这个案例不是写诗，也不是写创意文案，而是做数据提取和格式转换。

这种任务最重要的是：

- 稳定
- 准确
- 可复现

所以使用低 `temperature` 很合理。

### 工程视角

在工程系统里，如果你要做：

- 字段提取
- 格式转换
- 结构化输出
- 数据清洗

通常不希望模型每次都自由发挥。

你希望它像一个稳定的处理器，而不是一个灵感型创作者。

所以这里的设计意图是：

> 降低随机性，让 Prompt Chain 更可控。

---

## 9. 模块三：定义第一个 Prompt：提取信息

```python
prompt_extract = ChatPromptTemplate.from_template(
    "Extract the technical specifications from the following text:\n\n{text_input}"
)
```

这是第一步 Prompt。

它的任务很单一：

> 从输入文本中提取技术规格。

其中 `{text_input}` 是一个占位符。

运行时，代码会把真实文本填进去。

比如：

```text
Extract the technical specifications from the following text:

The new laptop model features a 3.5 GHz octa-core processor, 16GB of RAM, and a 1TB NVMe SSD.
```

### 这一段的业务意义

这一步不是让模型输出 JSON。

它只负责一件事：

> 把原始文本里的电脑规格找出来。

也就是从一段自然语言里找到：

- CPU：`3.5 GHz octa-core processor`
- Memory：`16GB of RAM`
- Storage：`1TB NVMe SSD`

### 接地气例子

想象你收到一条二手电脑广告：

```text
这台电脑速度很快，搭载 3.5GHz 八核处理器，16GB 内存，1TB 固态硬盘，适合剪视频。
```

第一步不是立刻整理成表格，而是先把“真正有用的配置”挑出来：

- `3.5GHz 八核处理器`
- `16GB 内存`
- `1TB 固态硬盘`

这就是 `prompt_extract` 的作用。

它像一个“信息筛选员”。

---

## 10. 模块四：定义第二个 Prompt：转换成 JSON

```python
prompt_transform = ChatPromptTemplate.from_template(
    "Transform the following specifications into a JSON object with 'cpu', 'memory', and 'storage' as keys:\n\n{specifications}"
)
```

这是第二步 Prompt。

它的任务是：

> 把第一步提取出来的规格，整理成 JSON。

这里的 `{specifications}` 也是一个占位符。

它不是用户直接传入的原始文本，而是第一步 `extraction_chain` 的输出。

这就是 Prompt Chaining 的关键。

### 这一段的业务意义

第一步得到的可能是普通文本：

```text
The laptop has a 3.5 GHz octa-core processor, 16GB of RAM, and a 1TB NVMe SSD.
```

第二步要把它变成结构化数据：

```json
{
  "cpu": "3.5 GHz octa-core processor",
  "memory": "16GB of RAM",
  "storage": "1TB NVMe SSD"
}
```

### 为什么要拆成第二步

因为“提取信息”和“格式转换”其实是两个不同任务。

- 提取信息：理解文本，找出有用内容。
- 格式转换：按照固定字段组织结果。

如果合在一起，模型既要理解，又要排版，还要保证 JSON 正确。

拆开之后，每一步都更简单。

### 工程视角

这一步体现的是第一章理论中的：

> The Role of Structured Output

JSON 的价值在于，它不是给人随便看看，而是给程序继续使用。

比如后续可以把结果存进数据库：

- `cpu` 字段 → 数据库 `cpu` 列
- `memory` 字段 → 数据库 `memory` 列
- `storage` 字段 → 数据库 `storage` 列

LangChain 官方文档也强调，结构化输出的意义是让模型返回特定、可预测的格式，例如 JSON、Pydantic model 或 dataclass，而不是只能得到一段自然语言。

---

## 11. 模块五：构建第一条 Chain

```python
extraction_chain = prompt_extract | llm | StrOutputParser()
```

这是第一段真正体现 LCEL 的代码。

它的意思是：

```text
prompt_extract
    ↓
llm
    ↓
StrOutputParser()
```

也就是：

1. 先把输入文本填进 Prompt。
2. 再交给模型处理。
3. 最后把模型输出转成字符串。

### 用普通 Python 思维理解

你可以把它想象成：

```python
formatted_prompt = prompt_extract.format(text_input=input_text)
model_response = llm.call(formatted_prompt)
specifications = parse_to_string(model_response)
```

但 LangChain 用 `|` 把它写成了更像流水线的形式：

```python
prompt_extract | llm | StrOutputParser()
```

### 这里的 `|` 是什么意思

在 LangChain LCEL 里，`|` 表示把左边的输出交给右边。

```text
A | B
```

意思接近：

```text
A 的结果 → 交给 B
```

LangChain 的官方参考中，`RunnableSequence` 是 LangChain 中最重要的组合操作之一，可以通过 `|` 操作符把多个 `Runnable` 串成顺序执行的链；一个 `Runnable` 的输出会作为下一个 `Runnable` 的输入。

所以这行代码就是：

```text
Prompt 模板 → 模型 → 字符串解析器
```

### 这一行为什么重要

因为它把第一步封装成了一个可以复用的小流程。

它不是只定义了一个 Prompt。

它定义了一个完整的处理单元：

- 输入：`text_input`
- 输出：`specifications`

这就是 Prompt Chaining 里的一个节点。

---

## 12. 模块六：构建完整 Chain

```python
full_chain = (
    {"specifications": extraction_chain}
    | prompt_transform
    | llm
    | StrOutputParser()
)
```

这是整段代码最关键的地方。

它把第一步和第二步接起来了。

### 先看这部分

```python
{"specifications": extraction_chain}
```

这句的意思是：

> 运行 `extraction_chain`，然后把它的输出放到 `specifications` 这个字段里。

而 `prompt_transform` 里面正好需要这个变量：

```text
{specifications}
```

所以它们刚好对上了。

### 数据流是这样的

用户输入：

```python
{"text_input": input_text}
```

先进入：

```python
extraction_chain
```

得到类似：

```text
3.5 GHz octa-core processor, 16GB of RAM, 1TB NVMe SSD
```

然后包装成：

```json
{
  "specifications": "3.5 GHz octa-core processor, 16GB of RAM, 1TB NVMe SSD"
}
```

再传给：

```python
prompt_transform
```

最终让模型转换成 JSON。

### 这就是 Prompt Chaining 的核心

这一段代码体现了整章最重要的一句话：

> 上一步的输出，成为下一步的输入。

也就是：

```text
text_input
    ↓
extraction_chain
    ↓
specifications
    ↓
prompt_transform
    ↓
final_result
```

### 接地气例子

想象你在做一个二手电脑信息录入系统。

用户发来一句话：

```text
这台电脑是 3.5GHz 八核处理器，16GB 内存，1TB SSD。
```

系统不能直接把这句话塞进数据库。

它要先做两步：

1. 识别里面有哪些配置。
2. 把配置填进固定字段。

类似：

```text
原始描述：
这台电脑是 3.5GHz 八核处理器，16GB 内存，1TB SSD。

提取结果：
3.5GHz 八核处理器 / 16GB 内存 / 1TB SSD

结构化结果：
cpu = 3.5GHz 八核处理器
memory = 16GB
storage = 1TB SSD
```

这就是业务逻辑。

代码只是把这个业务流程自动化了。

---

## 13. 模块七：准备输入文本

```python
input_text = "The new laptop model features a 3.5 GHz octa-core processor, 16GB of RAM, and a 1TB NVMe SSD."
```

这是原始输入。

它是一段自然语言，不是结构化数据。

也就是说，它不是这样：

```json
{
  "cpu": "3.5 GHz octa-core processor",
  "memory": "16GB of RAM",
  "storage": "1TB NVMe SSD"
}
```

而是这样：

```text
The new laptop model features a 3.5 GHz octa-core processor, 16GB of RAM, and a 1TB NVMe SSD.
```

所以这段代码要解决的问题就是：

> 如何把自然语言里的有用信息提取出来，并变成程序可用的数据。

这也是很多 AI 应用里非常常见的任务。

比如：

- 从简历中提取姓名、技能、工作年限。
- 从发票中提取金额、日期、公司名。
- 从商品描述中提取品牌、型号、价格。
- 从邮件中提取任务、截止时间、联系人。

---

## 14. 模块八：执行 Chain

```python
final_result = full_chain.invoke({"text_input": input_text})
```

这一行是真正运行整个流程。

这里传入的是：

```python
{"text_input": input_text}
```

### 为什么是字典

因为 Prompt 模板里有变量：

```text
{text_input}
```

所以调用时要告诉 Chain：

> `text_input` 对应的真实内容是什么？

完整执行过程是：

1. 把 `input_text` 填入 `prompt_extract`。
2. 调用 `llm` 提取规格。
3. 用 `StrOutputParser` 转成字符串。
4. 把这个字符串作为 `specifications`。
5. 填入 `prompt_transform`。
6. 再次调用 `llm`。
7. 再次用 `StrOutputParser` 转成字符串。
8. 得到 `final_result`。

这就是代码层面的完整链路。

---

## 15. 模块九：打印结果

```python
print("\n--- Final JSON Output ---")
print(final_result)
```

这两行只是把最终结果输出到终端。

可能看到类似：

```json
{
  "cpu": "3.5 GHz octa-core processor",
  "memory": "16GB of RAM",
  "storage": "1TB NVMe SSD"
}
```

注意：由于这里使用的是 `StrOutputParser()`，所以最终结果本质上仍然是一个字符串形式的 JSON，不是 Python 的 `dict`。

这点非常重要。

---

## 16. 一个容易被新手忽略的问题

原文代码里用了：

```python
StrOutputParser()
```

它只是把模型输出变成字符串。

也就是说，最终结果看起来像 JSON：

```json
{
  "cpu": "...",
  "memory": "...",
  "storage": "..."
}
```

但在 Python 里，它可能仍然是：

```python
str
```

而不是：

```python
dict
```

### 工程上这意味着什么

如果后续你要真正访问字段：

```python
result["cpu"]
```

可能不能直接用。

你可能还需要：

```python
import json

parsed_result = json.loads(final_result)
```

或者使用 LangChain 更严格的结构化输出能力。

这就是从 demo 走向工程系统时必须注意的地方。

LangChain 官方结构化输出文档也说明，结构化输出的核心价值是让应用直接得到可预测的数据结构，而不是只能解析自然语言文本。

---

## 17. 这段代码的核心骨架

如果把所有 `import`、注释、打印都去掉，这段代码的骨架其实是：

```python
llm = ChatOpenAI(temperature=0)

prompt_extract = ChatPromptTemplate.from_template(
    "Extract the technical specifications from the following text:\n\n{text_input}"
)

prompt_transform = ChatPromptTemplate.from_template(
    "Transform the following specifications into a JSON object with 'cpu', 'memory', and 'storage' as keys:\n\n{specifications}"
)

extraction_chain = prompt_extract | llm | StrOutputParser()

full_chain = (
    {"specifications": extraction_chain}
    | prompt_transform
    | llm
    | StrOutputParser()
)

final_result = full_chain.invoke({"text_input": input_text})
```

再压缩成逻辑表达就是：

```text
extract = prompt_extract → llm → string
transform = extract_output → prompt_transform → llm → string
```

最终一句话：

> 这段代码用 LCEL 把两个 LLM 调用串成了一条数据处理流水线。

---

## 18. 关键代码标亮版

这一章实战部分最值得标亮的是这几段。

### 关键 1：第一个 Prompt

```python
prompt_extract = ChatPromptTemplate.from_template(
    "Extract the technical specifications from the following text:\n\n{text_input}"
)
```

它体现：

> 第一步只负责信息提取。

### 关键 2：第二个 Prompt

```python
prompt_transform = ChatPromptTemplate.from_template(
    "Transform the following specifications into a JSON object with 'cpu', 'memory', and 'storage' as keys:\n\n{specifications}"
)
```

它体现：

> 第二步只负责格式转换。

### 关键 3：第一条 Chain

```python
extraction_chain = prompt_extract | llm | StrOutputParser()
```

它体现：

```text
Prompt → LLM → Parser
```

这是一个完整的 LLM 处理单元。

### 关键 4：完整 Chain

```python
full_chain = (
    {"specifications": extraction_chain}
    | prompt_transform
    | llm
    | StrOutputParser()
)
```

它体现：

> 第一步输出 → 第二步输入。

这是 Prompt Chaining 的核心。

### 关键 5：执行 Chain

```python
final_result = full_chain.invoke({"text_input": input_text})
```

它体现：

> 用一个输入触发整条链路。

用户只传入原始文本，后面所有步骤自动执行。

---

## 19. 这段代码背后的底层逻辑

### 19.1 PromptTemplate 的作用

`ChatPromptTemplate.from_template()` 做的事情是：

> 定义一个带变量的 Prompt 模板。

比如：

```text
Extract ... {text_input}
```

运行时会把：

```python
{"text_input": input_text}
```

填进去。

所以它的本质是：

> 把动态输入，变成完整 Prompt。

就像网页模板里的变量替换：

```html
<h1>Hello, {{ username }}</h1>
```

最后渲染成：

```html
<h1>Hello, Roy</h1>
```

PromptTemplate 的作用也类似：

```text
模板 + 用户输入 = 最终 Prompt
```

### 19.2 LLM 的作用

`llm` 就是真正进行语言理解和生成的组件。

在第一步里，它做：

> 从自然语言中识别电脑配置。

在第二步里，它做：

> 把配置改写成 JSON。

同一个模型，在不同 Prompt 下承担不同角色。

这也是 Prompt Chaining 的一个重要特点：

> 不是一定要用多个模型，而是可以用同一个模型，通过不同 Prompt 完成不同阶段的任务。

### 19.3 StrOutputParser 的作用

模型返回的通常不是普通字符串，而是一个消息对象。

`StrOutputParser()` 的作用是把它变成普通文本。

也就是：

```text
AIMessage → string
```

这样下一步才能继续使用。

如果没有 Parser，下游步骤拿到的可能不是简单文本，而是包含元数据的模型响应对象。

所以 Parser 是链条里的“格式清洗器”。

### 19.4 LCEL 的作用

LCEL 是 LangChain Expression Language。

这段代码用 LCEL 的 `|` 来连接组件：

```python
prompt_extract | llm | StrOutputParser()
```

它表达的是：

> 左边的输出，流向右边。

这和 Linux 管道很像：

```bash
cat file.txt | grep "cpu" | sort
```

每一步都处理上一步的结果。

LCEL 的价值在于让代码结构非常接近业务流程：

```text
Prompt → Model → Parser
```

而不是写成一堆临时变量和函数调用。

---

## 20. 从业务逻辑看这段代码

这段代码其实模拟了一个非常常见的业务需求：

> 把用户随口写的一段描述，变成系统可以保存和使用的结构化数据。

比如电商平台里，卖家写商品描述：

```text
这台电脑是 3.5GHz 八核处理器，16GB 内存，1TB SSD。
```

系统希望自动生成：

```json
{
  "cpu": "3.5GHz 八核处理器",
  "memory": "16GB",
  "storage": "1TB SSD"
}
```

这样后续就可以：

- 做筛选
- 做搜索
- 进数据库
- 做商品对比
- 给用户推荐

所以这段代码不是玩具代码。

它背后对应的是非常真实的 AI 应用场景：

```text
非结构化文本 → 信息提取 → 结构化数据
```

---

## 21. 从 Prompt Chaining 主题看这段代码

这段代码刚好对应本章三个核心概念。

### 21.1 对应 Limitations of Single Prompts

如果用一个 Prompt 一次做完：

```text
请提取规格，并输出 JSON。
```

可能出现：

- 字段漏掉。
- JSON 不规范。
- 把无关信息也放进去。
- 输出解释文字混在 JSON 外面。

所以单个 Prompt 有不稳定性。

### 21.2 对应 Sequential Decomposition

代码把任务拆成两步：

```text
Step 1：Extract Information
Step 2：Transform to JSON
```

每一步只做一件事。

这降低了模型负担。

### 21.3 对应 Structured Output

第二步明确要求：

```text
with 'cpu', 'memory', and 'storage' as keys
```

这就是在约束输出结构。

不过原文代码只是要求模型“输出 JSON”，并没有做严格 schema 校验。

所以它是一个入门级结构化输出示例。

---

## 22. 工程视角：这段代码有什么优点

### 优点一：流程清晰

```text
提取 → 转换
```

非常直观。

### 优点二：模块化

第一步可以单独测试：

```python
extraction_chain.invoke({"text_input": input_text})
```

第二步也可以单独测试。

这对调试很重要。

### 优点三：可扩展

后续可以继续加步骤：

```text
提取规格
    ↓
转换 JSON
    ↓
校验字段
    ↓
补全缺失值
    ↓
存入数据库
```

这就是 Prompt Chain 的扩展能力。

### 优点四：更贴近真实系统

真实业务系统不会只是“问模型一句话”。

它往往是：

- 输入处理
- 模型调用
- 结果解析
- 校验
- 转换
- 存储

这段代码虽然简单，但已经展示了这种工程结构。

---

## 23. 工程视角：这段代码有什么不足

这部分很重要，因为读书笔记不能只说优点。

### 不足一：JSON 没有被严格校验

代码用了：

```python
StrOutputParser()
```

所以最终只是字符串。

如果模型输出：

```text
Here is the JSON:
{
  "cpu": "...",
  "memory": "...",
  "storage": "..."
}
```

这对人来说能看懂，但对程序来说可能不是合法 JSON。

更工程化的做法是使用：

- JSON parser
- Pydantic schema
- structured output

这样可以真正约束字段。

### 不足二：没有错误处理

如果模型返回格式不对，代码没有处理。

真实项目里可能需要：

- `try/except`
- 重试机制
- 格式校验
- fallback prompt
- 日志记录

### 不足三：没有检查字段是否缺失

如果输入文本里没有 `storage` 信息，模型可能会猜一个。

真实系统里应该区分：

- 字段不存在
- 字段不确定
- 字段由模型推测

比如更安全的 JSON 应该允许：

```json
{
  "cpu": "3.5 GHz octa-core processor",
  "memory": "16GB of RAM",
  "storage": null
}
```

而不是让模型乱补。

### 不足四：没有业务校验

比如：

- `memory` 是否符合常见容量格式？
- `storage` 是否是硬盘容量？
- `cpu` 是否真的像处理器描述？

这些最好由确定性代码或规则系统来检查，而不是完全依赖 LLM。

---

## 24. 如果我要把它改成更工程化，会怎么做

原文代码适合作为教学示例。

如果是实际项目，我会把它升级成这样：

1. 输入文本。
2. LLM 提取规格。
3. LLM 输出结构化 JSON。
4. JSON parser 解析。
5. Pydantic 校验字段。
6. 如果失败，触发重试或修复 Prompt。
7. 输出 Python `dict`。
8. 存入数据库或返回 API。

也就是：

```text
LLM 负责理解
Parser 负责解析
Schema 负责校验
业务代码负责兜底
```

这才是更可靠的工程方案。

---

## 25. 新手最应该看懂的 3 行代码

如果新手只记住三行，应该记住这三行。

第一行：

```python
extraction_chain = prompt_extract | llm | StrOutputParser()
```

它表示：

> 把原始文本变成提取后的规格。

第二行：

```python
full_chain = (
    {"specifications": extraction_chain}
    | prompt_transform
    | llm
    | StrOutputParser()
)
```

它表示：

> 把第一步的输出作为第二步的输入。

第三行：

```python
final_result = full_chain.invoke({"text_input": input_text})
```

它表示：

> 用原始输入启动整条链。

这三行就是整个实战案例的骨架。

---

## 26. 一句话总结这段代码

这段代码用 LangChain LCEL 实现了一个最基础的 Prompt Chain：

> 第一步从自然语言中提取电脑规格，第二步把提取结果转换成 JSON，从而把一个复杂任务拆成两个更稳定、更可控的处理阶段。

它的重点不是“用了 LangChain”，而是展示了 Prompt Chaining 的工程本质：

```text
复杂任务
    ↓
拆成小步骤
    ↓
每一步只做一件事
    ↓
上一步输出传给下一步
    ↓
最终得到更可靠的结果
```

---

## 27. 发布版小结

这段代码展示了 Prompt Chaining 最小可行实现。

它不是让模型一次性完成所有任务，而是把“提取信息”和“格式转换”拆成两个连续步骤。

第一条链负责从文本中提取技术规格，第二条链负责把规格转换成 JSON。

其中 `|` 表示数据从左到右流动，`extraction_chain` 的输出会被放入 `specifications`，再交给第二个 Prompt 使用。

从工程视角看，这种写法让流程更清晰、模块更容易测试，也为后续加入字段校验、错误处理、数据库存储打下基础。
