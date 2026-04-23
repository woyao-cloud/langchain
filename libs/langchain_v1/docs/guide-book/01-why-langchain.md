# 第一章：为什么需要 LangChain

## 大语言模型的局限

大语言模型（LLM）能力惊人，但单独使用时存在根本性局限：

### 知识截止与幻觉

模型的知识停留在训练数据的截止日期。当被问到训练后发生的事，它要么坦承不知道，要么编造看似合理的答案（即"幻觉"）。

```python
# 模型可能自信地给出错误答案
response = model.invoke("2024年诺贝尔物理学奖得主是谁？")
# 可能产生幻觉——给出看似合理但错误的信息
```

### 无法与外部世界交互

模型被困在文本世界里，无法查询数据库、调用 API、读取文件或执行计算：

```python
# 模型无法做到这些：
# - 查询当前天气
# - 读取公司内部数据库
# - 执行精确的数学计算
# - 调用外部 API
```

### 上下文窗口有限

即使是最大的模型，也只能处理有限长度的输入。长对话、长文档、大量工具描述都会快速耗尽上下文窗口。

### 缺乏结构化输出

原始模型输出是自由文本，但应用通常需要结构化数据：

```python
# 我们想要的
{"name": "张三", "age": 25, "email": "zhangsan@example.com"}

# 模型通常给出的
"这个用户叫张三，25岁，邮箱是 zhangsan@example.com"
```

## LangChain 解决什么问题

LangChain 正是为解决上述局限而设计的。它不是一个模型，而是一个让模型变得更有用的框架。

### 核心定位

**LangChain 是连接大语言模型与现实世界的桥梁。**

```
┌─────────────┐     LangChain      ┌─────────────┐
│   大语言模型  │ ◄──────────────► │   现实世界    │
│  (推理引擎)  │                    │ (数据/工具/API)│
└─────────────┘                    └─────────────┘
```

### 六大问题域

| 问题 | LangChain 的解决方案 |
|------|---------------------|
| 知识截止 | **RAG（检索增强生成）**：先检索外部知识，再让模型基于检索结果回答 |
| 无法交互 | **工具调用**：让模型调用搜索、计算、数据库等外部工具 |
| 上下文有限 | **对话管理与摘要**：自动管理上下文窗口，摘要历史消息 |
| 输出无结构 | **结构化输出**：强制模型返回 Pydantic 模型、TypedDict 等结构化数据 |
| 单次调用不够 | **智能体循环**：模型→工具→模型→工具的迭代推理，直到任务完成 |
| 调试困难 | **可观测性**：与 LangSmith 集成，追踪每一步的输入输出 |

### 设计哲学

LangChain 遵循三个核心设计原则：

**1. 组合优于继承**

每个能力都是可插拔的组件。你可以自由组合模型、工具、中间件：

```python
# 组合不同的能力
agent = create_agent(
    model,                                    # 选择模型
    tools=[search, calculate],                # 选择工具
    middleware=[retry, pii_detection],        # 选择中间件
    response_format=AutoStrategy(UserInfo),   # 选择输出格式
)
```

**2. 声明式优于命令式**

用几行代码声明你想要什么，而不是写大量胶水代码：

```python
# 声明式：告诉 LangChain 你要什么
agent = create_agent("anthropic:claude-sonnet-4-20250514", tools=[...])

# 命令式（没有 LangChain 的情况下）：
# 1. 初始化模型客户端
# 2. 构造 prompt
# 3. 解析模型输出
# 4. 如果是工具调用，执行工具
# 5. 把工具结果喂回模型
# 6. 循环直到模型不再调用工具
# 7. 解析最终输出
```

**3. 渐进式复杂度**

简单的事情简单做，复杂的事情也能做：

```python
# 最简用法——5行代码创建智能体
agent = create_agent(model)
result = agent.invoke({"messages": [("user", "你好")]})

# 复杂用法——完整的中间件栈
agent = create_agent(
    model,
    tools=[...],
    middleware=[
        HumanInTheLoopMiddleware(interrupt_on={"send_email": True}),
        ModelRetryMiddleware(max_retries=3),
        ModelCallLimitMiddleware(run_limit=20),
        PIIMiddleware("email", strategy="redact"),
    ],
    checkpointer=MemorySaver(),
    response_format=AutoStrategy(Report),
)
```

## LangChain 与 LangGraph 的关系

LangChain 生态由两个核心框架组成：

| 框架 | 定位 | 适用场景 |
|------|------|---------|
| **LangChain** | 高层智能体框架 | 快速构建智能体应用 |
| **LangGraph** | 底层工作流引擎 | 精细控制复杂工作流 |

LangChain 的智能体内部构建在 LangGraph 之上。大多数场景下，直接使用 LangChain 即可。当你需要：

- 自定义复杂的状态图
- 精确控制每一步的执行逻辑
- 构建非智能体的工作流

这时可以降级到 LangGraph 层。

```python
# LangChain：高层 API，快速上手
from langchain.agents import create_agent
agent = create_agent(model, tools=[...])

# LangGraph：底层 API，精细控制
from langgraph.graph import StateGraph
graph = StateGraph(MyState)
graph.add_node("think", think_node)
graph.add_node("act", act_node)
graph.add_edge("think", "act")
```

## LangSmith：可观测性

LangChain 与 LangSmith 深度集成，提供：

- **追踪**：每一步的输入输出、延迟、token 用量
- **测试**：自动化评估智能体输出质量
- **标注**：人工标注数据集用于微调
- **部署**：一键部署智能体到生产环境

```python
# 只需设置环境变量即可启用追踪
import os
os.environ["LANGSMITH_API_KEY"] = "your-key"
os.environ["LANGSMITH_TRACING"] = "true"
```

## 小结

| 概念 | 关键点 |
|------|--------|
| LLM 局限 | 知识截止、无法交互、上下文有限、输出无结构 |
| LangChain 定位 | 连接 LLM 与现实世界的桥梁 |
| 核心能力 | 工具调用、RAG、结构化输出、智能体循环、中间件 |
| 设计哲学 | 组合性、声明式、渐进式复杂度 |
| 生态 | LangChain（高层）+ LangGraph（底层）+ LangSmith（可观测） |

下一章我们探讨支撑这些能力的理论基础。