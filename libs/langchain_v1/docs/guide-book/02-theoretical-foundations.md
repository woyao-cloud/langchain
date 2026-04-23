# 第二章：理论支撑

LangChain 的设计并非凭空而来，而是建立在一系列经过验证的 AI 研究基础之上。理解这些理论，能帮你更好地使用框架。

## ReAct：推理与行动交织

ReAct（Reasoning + Acting）是 LangChain 智能体的核心理论框架，由 Yao 等人于 2022 年提出。

### 核心思想

传统方法将推理和行动分离——要么纯推理（链式思考），要么纯行动（行为主义）。ReAct 让模型交替进行推理和行动：

```
思考：用户问的是北京天气，我需要查询天气信息
行动：调用 search_weather("北京")
观察：北京：晴天，气温 25°C
思考：已获得天气信息，可以回答用户了
回答：北京今天天气晴朗，气温 25°C
```

### 为什么有效

1. **推理指导行动**：模型先想清楚要做什么，再执行，减少盲目尝试
2. **行动验证推理**：工具返回的真实结果帮助模型修正错误推理
3. **可追踪性**：每一步推理和行动都有记录，便于调试

### 在 LangChain 中的体现

`create_agent()` 创建的智能体本质上就是一个 ReAct 循环：

```python
agent = create_agent(model, tools=[search_weather])
result = agent.invoke({"messages": [("user", "北京天气怎么样？")]})

# 内部执行过程：
# 1. 模型推理 → 需要调用 search_weather
# 2. 工具执行 → 返回天气数据
# 3. 模型推理 → 基于数据生成回答
# 4. 循环结束 → 返回结果
```

## 链式思考（Chain-of-Thought）

链式思考（CoT）由 Wei 等人于 2022 年提出，是 ReAct 推理部分的基础。

### 核心思想

让模型在给出答案前展示推理步骤，显著提升复杂推理任务的准确率：

```python
# 不用 CoT（容易出错）
问：一个商店有 23 个苹果，卖了 15 个，又进货了 8 个。现在有多少？
答：16

# 用 CoT（更准确）
问：一个商店有 23 个苹果，卖了 15 个，又进货了 8 个。现在有多少？
思考：开始有 23 个，卖了 15 个剩下 23 - 15 = 8 个，又进货了 8 个所以是 8 + 8 = 16 个
答：16
```

### 在 LangChain 中的体现

- `ReasoningContentBlock`：一些模型支持思维链输出，LangChain 可以接收
- 系统提示词：通过 `system_prompt` 引导模型展示推理过程
- 中间件钩子：`before_model` 可以注入 CoT 提示

```python
# 通过系统提示引导 CoT
agent = create_agent(
    model,
    system_prompt="请先展示推理步骤，再给出最终答案。",
)
```

## 检索增强生成（RAG）

RAG（Retrieval-Augmented Generation）由 Lewis 等人于 2020 年提出，是解决知识截止问题的核心方法。

### 核心思想

不要让模型凭记忆回答，而是先从外部知识库检索相关信息，再基于检索结果生成答案：

```
用户提问
   ↓
检索相关文档（从向量数据库）
   ↓
将文档作为上下文喂给模型
   ↓
模型基于上下文生成答案
```

### 为什么有效

1. **知识可更新**：只需更新知识库，无需重新训练模型
2. **减少幻觉**：模型基于真实文档回答，而非凭空编造
3. **可验证**：可以溯源答案来自哪个文档
4. **节省成本**：避免为每个知识更新都微调模型

### 在 LangChain 中的体现

```python
from langchain.embeddings import init_embeddings
from langchain.agents import create_agent

# 嵌入模型用于文档向量化
embeddings = init_embeddings("openai:text-embedding-3-small")

# 智能体可以配合向量数据库实现 RAG
# （向量数据库如 FAISS、Chroma 等需要额外安装）
```

## 工具使用（Tool Use）

工具使用能力是大语言模型从"文本生成器"进化为"行动者"的关键。

### 核心思想

模型不仅能生成文本，还能决定何时调用外部工具，以及传递什么参数：

```json
// 模型输出不是纯文本，而是结构化的工具调用
{
  "tool_calls": [
    {
      "name": "search_weather",
      "arguments": {"city": "北京"}
    }
  ]
}
```

### 为什么有效

1. **扩展能力边界**：模型无需"知道"一切，只需知道"如何查找"
2. **精确执行**：计算、数据库查询等交给专门的工具，而非让模型"猜测"
3. **与现有系统集成**：通过工具桥接模型和企业系统

### 在 LangChain 中的体现

```python
from langchain_core.tools import tool

@tool
def calculate(expression: str) -> float:
    """计算数学表达式。"""
    return eval(expression)

agent = create_agent(model, tools=[calculate])
# 模型会自动判断何时调用 calculate
```

## 多智能体系统（Multi-Agent）

多智能体系统是智能体架构的自然延伸——让多个专业化的智能体协作完成复杂任务。

### 核心思想

不是用一个"万能"智能体，而是让多个专精不同领域的智能体分工协作：

```
用户请求
   ↓
协调者智能体 → 分配给专家
   ↓
搜索专家 → 找信息
计算专家 → 做计算
写作专家 → 写报告
   ↓
协调者整合 → 返回结果
```

### 在 LangChain 中的体现

LangChain v1 目前聚焦于单智能体架构。多智能体编排需要使用 LangGraph：

```python
# LangGraph 多智能体编排（高级用法）
from langgraph.graph import StateGraph

# 每个智能体是一个图节点
graph = StateGraph(OverallState)
graph.add_node("researcher", researcher_agent)
graph.add_node("writer", writer_agent)
graph.add_node("reviewer", reviewer_agent)
```

## 人在回路（Human-in-the-Loop）

人在回路不是纯理论，而是一个重要的工程原则：在高风险操作前暂停，等待人类审批。

### 核心思想

AI 系统不应该是完全自动的"黑箱"。在关键决策点，应该有人类介入：

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware

# 发送邮件前需要人工审批
hitl = HumanInTheLoopMiddleware(
    interrupt_on={"send_email": True}
)
```

### 为什么重要

1. **安全性**：防止 AI 执行不可逆操作（删除数据、发送邮件、转账）
2. **合规性**：许多行业要求关键操作有人类审批
3. **信任**：用户更信任需要他们确认的系统
4. **调试**：审批点提供了检查 AI 推理的窗口

## 上下文管理（Context Management）

上下文管理是工程实践中的核心挑战——如何在有限的上下文窗口中容纳必要信息。

### 核心挑战

- 上下文窗口有上限（如 128K tokens）
- 长对话会逐渐耗尽窗口
- 工具调用结果会累积
- 系统提示占用窗口空间

### LangChain 的解决方案

```python
# 方案一：摘要中间件——自动压缩历史
from langchain.agents.middleware import SummarizationMiddleware

summary = SummarizationMiddleware(
    model="anthropic:claude-sonnet-4-20250514",
    trigger=("messages", 50),  # 超过 50 条消息时触发摘要
    keep=("messages", 20),      # 保留最近 20 条
)

# 方案二：上下文编辑——裁剪旧的工具结果
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit

editor = ContextEditingMiddleware(
    edits=[ClearToolUsesEdit(trigger=100_000, keep=3)],
)
```

## 小结

| 理论 | 核心思想 | LangChain 中的体现 |
|------|---------|-------------------|
| ReAct | 推理与行动交织 | `create_agent()` 的执行循环 |
| CoT | 展示推理步骤 | `system_prompt`、`ReasoningContentBlock` |
| RAG | 检索增强生成 | `init_embeddings()`、向量数据库集成 |
| 工具使用 | 模型调用外部工具 | `@tool`、`ToolNode`、工具选择中间件 |
| 多智能体 | 专家协作 | LangGraph 多智能体编排 |
| 人在回路 | 关键决策需人工审批 | `HumanInTheLoopMiddleware` |
| 上下文管理 | 有限窗口的信息优化 | `SummarizationMiddleware`、`ContextEditingMiddleware` |

这些理论不是孤立的——它们在 LangChain 中交织在一起，共同构成了智能体应用的基石。下一章我们将看到这些理论如何转化为真实的使用场景。