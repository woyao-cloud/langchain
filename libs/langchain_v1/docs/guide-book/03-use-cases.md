# 第三章：使用场景

理论只有在解决真实问题时才有价值。本章展示 LangChain 在七大典型场景中的应用。

## 场景一：智能客服

### 问题描述

传统客服系统依赖关键词匹配，无法理解复杂问题，且无法查询实时数据。

### LangChain 方案

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.agents.middleware import (
    HumanInTheLoopMiddleware,
    ContextEditingMiddleware,
    ClearToolUsesEdit,
)
from langchain_core.tools import tool

@tool
def query_order(order_id: str) -> str:
    """查询订单状态。"""
    # 连接真实订单系统
    return f"订单 {order_id}：已发货，预计明天送达"

@tool
def check_inventory(product_id: str) -> str:
    """查询商品库存。"""
    return f"商品 {product_id}：库存充足，全国 3 个仓库有货"

@tool
def create_return(order_id: str, reason: str) -> str:
    """创建退货申请。"""
    return f"已为订单 {order_id} 创建退货申请，原因：{reason}"

model = init_chat_model("anthropic:claude-sonnet-4-20250514")
agent = create_agent(
    model,
    tools=[query_order, check_inventory, create_return],
    system_prompt="你是一个专业的客服助手。请用礼貌、准确的语言回答客户问题。",
    middleware=[
        # 退货操作需要人工确认
        HumanInTheLoopMiddleware(interrupt_on={"create_return": True}),
        # 长对话自动压缩
        ContextEditingMiddleware(
            edits=[ClearToolUsesEdit(trigger=50_000, keep=3)],
        ),
    ],
)
```

### 关键能力

- **工具调用**：查询订单、库存等实时数据
- **人在回路**：退货等高风险操作需审批
- **上下文管理**：长对话自动压缩，不会超出窗口

## 场景二：知识库问答（RAG）

### 问题描述

企业拥有大量内部文档，员工查找信息效率低下，且容易遗漏重要内容。

### LangChain 方案

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.embeddings import init_embeddings
from langchain_core.tools import tool

# 初始化嵌入模型（用于文档向量化）
embeddings = init_embeddings("openai:text-embedding-3-small")

@tool
def search_knowledge_base(query: str) -> str:
    """在企业知识库中搜索相关信息。"""
    # 实际实现会使用向量数据库（如 FAISS、Chroma）
    # 这里展示核心流程
    results = vector_db.similarity_search(query, k=3)
    return "\n".join([r.page_content for r in results])

model = init_chat_model("anthropic:claude-sonnet-4-20250514")
agent = create_agent(
    model,
    tools=[search_knowledge_base],
    system_prompt=(
        "你是一个企业知识库助手。根据搜索到的信息回答问题。"
        "如果搜索结果中没有相关信息，请诚实地说不知道。"
        "回答时请引用信息来源。"
    ),
)
```

### 关键能力

- **RAG**：先检索，再生成，避免幻觉
- **可溯源**：回答基于具体文档，可验证

## 场景三：数据分析助手

### 问题描述

业务人员需要从数据库提取洞察，但不熟悉 SQL 或 Python。

### LangChain 方案

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.agents.middleware import ModelCallLimitMiddleware
from langchain.agents.structured_output import AutoStrategy
from pydantic import BaseModel, Field
from langchain_core.tools import tool

@tool
def execute_sql(query: str) -> str:
    """执行 SQL 查询并返回结果。只允许 SELECT 语句。"""
    if not query.strip().upper().startswith("SELECT"):
        return "错误：只允许 SELECT 查询"
    # 执行查询
    return db.execute(query)

@tool
def create_chart(data_json: str, chart_type: str, title: str) -> str:
    """根据数据创建图表。"""
    # 生成图表
    return f"图表已创建：{title} ({chart_type})"

class AnalysisResult(BaseModel):
    """数据分析结果。"""
    summary: str = Field(description="分析摘要")
    key_findings: list[str] = Field(description="关键发现")
    recommendation: str = Field(description="建议")

agent = create_agent(
    model,
    tools=[execute_sql, create_chart],
    response_format=AutoStrategy(AnalysisResult),
    middleware=[
        # 限制模型调用次数，防止无限循环
        ModelCallLimitMiddleware(run_limit=15),
    ],
    system_prompt="你是一个数据分析助手。帮助用户查询数据并生成分析报告。",
)
```

### 关键能力

- **工具调用**：安全地执行 SQL 查询
- **结构化输出**：返回格式化的分析结果
- **安全限制**：限制调用次数，防止无限循环

## 场景四：代码生成与执行

### 问题描述

需要 AI 辅助编写代码，并在安全环境中执行验证。

### LangChain 方案

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.agents.middleware import (
    ShellToolMiddleware,
    DockerExecutionPolicy,
    HumanInTheLoopMiddleware,
)

agent = create_agent(
    model,
    # ShellToolMiddleware 提供 shell 工具
    middleware=[
        ShellToolMiddleware(
            workspace_root="/workspace",
            execution_policy=DockerExecutionPolicy(image="python:3.12"),
        ),
        # 执行前需确认
        HumanInTheLoopMiddleware(
            interrupt_on={"shell": {"allowed_decisions": ["approve", "reject"]}},
        ),
    ],
    system_prompt="你是一个编程助手。编写代码前先理解需求，执行前请用户确认。",
)
```

### 关键能力

- **安全执行**：Docker 容器隔离
- **人在回路**：执行前需人类确认
- **PII 保护**：可选加上 `PIIMiddleware` 防止代码中泄露敏感信息

## 场景五：内容创作与审核

### 问题描述

批量生成内容（文章、摘要、翻译），同时确保内容质量和合规性。

### LangChain 方案

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.agents.middleware import (
    PIIMiddleware,
    ModelRetryMiddleware,
)
from langchain.agents.structured_output import AutoStrategy
from pydantic import BaseModel, Field

class Article(BaseModel):
    """文章结构。"""
    title: str = Field(description="标题")
    content: str = Field(description="正文")
    tags: list[str] = Field(description="标签列表")

agent = create_agent(
    model,
    response_format=AutoStrategy(Article),
    middleware=[
        # 检测并脱敏 PII
        PIIMiddleware("email", strategy="mask"),
        PIIMiddleware("credit_card", strategy="block"),
        # 模型调用重试
        ModelRetryMiddleware(max_retries=2),
    ],
    system_prompt="你是一个内容创作助手。根据要求生成文章，确保内容合规。",
)
```

### 关键能力

- **结构化输出**：返回结构化的文章对象
- **PII 检测**：自动脱敏敏感信息
- **重试机制**：模型调用失败自动重试

## 场景六：多步骤工作流

### 问题描述

复杂任务需要多个步骤按顺序执行，每步依赖前一步的结果。

### LangChain 方案

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.agents.middleware import TodoListMiddleware

agent = create_agent(
    model,
    tools=[search_tool, write_tool, review_tool],
    middleware=[
        # 任务列表中间件帮助跟踪进度
        TodoListMiddleware(),
    ],
    system_prompt=(
        "你是一个项目管理助手。收到任务后："
        "1. 用 write_todos 创建任务列表"
        "2. 按顺序执行每一步"
        "3. 完成后标记任务完成"
    ),
)
```

### 关键能力

- **任务追踪**：`TodoListMiddleware` 维护任务列表
- **多步执行**：智能体自动按步骤推进
- **可见进度**：用户可以看到任务完成状态

## 场景七：智能路由与模型回退

### 问题描述

不同复杂度的问题需要不同能力的模型，且主要模型可能暂时不可用。

### LangChain 方案

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.agents.middleware import ModelFallbackMiddleware

# 模型回退：主模型失败时切换到备用
agent = create_agent(
    model,
    middleware=[
        ModelFallbackMiddleware(
            "anthropic:claude-sonnet-4-20250514",  # 第一备用
            "openai:gpt-4o",                       # 第二备用
        ),
    ],
)
```

### 关键能力

- **模型回退**：主模型不可用时自动切换
- **成本优化**：简单问题用小模型，复杂问题用大模型

## 场景选择指南

| 你的需求 | 推荐场景 | 核心中间件 |
|---------|---------|-----------|
| 回答基于实时数据的问题 | 场景一（智能客服） | `HumanInTheLoopMiddleware` |
| 从大量文档中查找信息 | 场景二（RAG） | `ContextEditingMiddleware` |
| 安全地查询和分析数据 | 场景三（数据分析） | `ModelCallLimitMiddleware` |
| 生成并执行代码 | 场景四（代码执行） | `ShellToolMiddleware` |
| 批量生成合规内容 | 场景五（内容创作） | `PIIMiddleware` |
| 管理多步骤任务 | 场景六（多步工作流） | `TodoListMiddleware` |
| 保证服务可用性 | 场景七（模型回退） | `ModelFallbackMiddleware` |

下一章将深入每个 API 的详细用法。