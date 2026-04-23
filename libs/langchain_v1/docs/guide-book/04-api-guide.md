# 第四章：API 使用指南

本章是 LangChain v1 核心 API 的实战指南。每个 API 都从"为什么需要"出发，配合可运行的代码示例。

## 初始化聊天模型

### 为什么需要 init_chat_model()

不同的模型提供商有不同的初始化方式。`init_chat_model()` 统一了所有提供商的接口：

```python
from langchain.chat_models import init_chat_model

# 不用 init_chat_model —— 每个提供商需要不同的导入和初始化
from langchain_anthropic import ChatAnthropic
from langchain_openai import ChatOpenAI
model_a = ChatAnthropic(model="claude-sonnet-4-20250514")
model_b = ChatOpenAI(model="gpt-4o")

# 用 init_chat_model —— 统一接口
model_a = init_chat_model("claude-sonnet-4-20250514", model_provider="anthropic")
model_b = init_chat_model("gpt-4o", model_provider="openai")
```

### 基本用法

```python
# 方式一：显式指定提供商（推荐）
model = init_chat_model("claude-sonnet-4-20250514", model_provider="anthropic")

# 方式二：前缀指定提供商
model = init_chat_model("anthropic:claude-sonnet-4-20250514")

# 方式三：自动推断（部分模型支持）
model = init_chat_model("gpt-4o")  # 自动识别为 openai
```

### 运行时切换模型

```python
# 创建可配置模型
configurable_model = init_chat_model(
    configurable_fields=["model", "model_provider"],
)

# 运行时选择模型
result = configurable_model.invoke(
    "你好",
    config={"configurable": {"model": "gpt-4o", "model_provider": "openai"}}
)
```

### 传递额外参数

```python
# 传递模型特定参数（如温度）
model = init_chat_model(
    "claude-sonnet-4-20250514",
    model_provider="anthropic",
    temperature=0.7,       # 控制随机性
    max_tokens=1024,      # 限制输出长度
)
```

## 创建智能体

### 最简创建

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

model = init_chat_model("anthropic:claude-sonnet-4-20250514")
agent = create_agent(model)
```

### 带工具创建

```python
from langchain_core.tools import tool

@tool
def add(a: int, b: int) -> int:
    """两数相加。"""
    return a + b

@tool
def multiply(a: int, b: int) -> int:
    """两数相乘。"""
    return a * b

agent = create_agent(model, tools=[add, multiply])
```

### 带系统提示创建

```python
from langchain_core.messages import SystemMessage

agent = create_agent(
    model,
    system_prompt="你是一个数学助手。只回答数学问题。",
)

# 或使用 SystemMessage 对象
agent = create_agent(
    model,
    system_prompt=SystemMessage(content="你是一个数学助手。只回答数学问题。"),
)
```

### 带中间件创建

```python
from langchain.agents.middleware import (
    HumanInTheLoopMiddleware,
    ModelRetryMiddleware,
    ModelCallLimitMiddleware,
)

agent = create_agent(
    model,
    tools=[add, multiply],
    middleware=[
        HumanInTheLoopMiddleware(interrupt_on={"multiply": True}),
        ModelRetryMiddleware(max_retries=2),
        ModelCallLimitMiddleware(run_limit=10),
    ],
)
```

### 带结构化输出创建

```python
from pydantic import BaseModel, Field
from langchain.agents.structured_output import AutoStrategy

class MathResult(BaseModel):
    expression: str = Field(description="数学表达式")
    result: float = Field(description="计算结果")
    explanation: str = Field(description="计算过程说明")

agent = create_agent(
    model,
    tools=[add, multiply],
    response_format=AutoStrategy(MathResult),
)
```

### 带持久化创建

```python
from langgraph.checkpoint.memory import MemorySaver

agent = create_agent(
    model,
    checkpointer=MemorySaver(),  # 内存持久化
)

# 使用 thread_id 维持对话
config = {"configurable": {"thread_id": "user-123"}}
result1 = agent.invoke({"messages": [("user", "我叫小明")]}, config=config)
result2 = agent.invoke({"messages": [("user", "我叫什么？")]}, config=config)
# 智能体会记住"小明"
```

## 调用智能体

### 同步调用

```python
result = agent.invoke({"messages": [("user", "3加5等于几？")]})
print(result["messages"][-1].content)
```

### 流式调用

```python
for chunk in agent.stream({"messages": [("user", "讲一个故事")]}, config=config):
    print(chunk, end="", flush=True)
```

### 异步调用

```python
result = await agent.ainvoke({"messages": [("user", "你好")]})

async for chunk in agent.astream({"messages": [("user", "讲一个故事")]}, config=config):
    print(chunk, end="", flush=True)
```

## 定义工具

### @tool 装饰器（最常用）

```python
from langchain_core.tools import tool

@tool
def search(query: str, max_results: int = 5) -> list[str]:
    """搜索互联网上的信息。

    Args:
        query: 搜索关键词
        max_results: 最大返回数量，默认 5
    """
    return [f"结果{i}: {query}" for i in range(max_results)]
```

### StructuredTool（精细控制）

```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel, Field

class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")
    max_results: int = Field(default=5, description="最大返回数量")

def search_func(query: str, max_results: int = 5) -> list[str]:
    return [f"结果{i}: {query}" for i in range(max_results)]

search_tool = StructuredTool.from_function(
    func=search_func,
    name="search",
    description="搜索互联网上的信息",
    args_schema=SearchInput,
)
```

### 访问智能体状态

```python
from langchain.tools import InjectedState
from langchain_core.tools import tool
from typing import Annotated

@tool
def count_messages(state: Annotated[dict, InjectedState]) -> str:
    """返回当前对话的消息数量。"""
    return f"对话中有 {len(state['messages'])} 条消息"
```

## 中间件实战

### 重试与回退

```python
from langchain.agents.middleware import ModelRetryMiddleware, ModelFallbackMiddleware

agent = create_agent(
    model,
    middleware=[
        # 失败时自动重试
        ModelRetryMiddleware(
            max_retries=2,
            retry_on=(TimeoutError, RateLimitError),
        ),
        # 重试仍失败时回退到备用模型
        ModelFallbackMiddleware("openai:gpt-4o"),
    ],
)
```

### 人工审批

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware

agent = create_agent(
    model,
    tools=[send_email, delete_file],
    middleware=[
        # 发邮件和删文件需要人工确认
        HumanInTheLoopMiddleware(
            interrupt_on={
                "send_email": {"allowed_decisions": ["approve", "reject"]},
                "delete_file": {"allowed_decisions": ["approve", "reject"]},
            },
        ),
    ],
)
```

### PII 保护

```python
from langchain.agents.middleware import PIIMiddleware

agent = create_agent(
    model,
    middleware=[
        PIIMiddleware("email", strategy="mask"),        # 遮掩邮箱
        PIIMiddleware("credit_card", strategy="block"),  # 阻止信用卡号
        PIIMiddleware("phone", strategy="redact"),       # 脱敏电话号码
    ],
)
```

### 自定义中间件

```python
from langchain.agents.middleware import before_model, wrap_model_call
from langchain_core.messages import SystemMessage

# 用装饰器快速定义
@before_model
def add_timestamp(state, runtime):
    """在每步开始时注入当前时间信息。"""
    from datetime import datetime
    timestamp = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
    return {"system_hint": f"当前时间：{timestamp}"}

# 用 wrap_model_call 修改请求
@wrap_model_call
def add_language_hint(request, handler):
    """在系统消息中添加语言提示。"""
    hint = "请用中文回答。"
    if request.system_message:
        new_system = request.system_message.content + "\n" + hint
        new_request = request.override(system_message=SystemMessage(content=new_system))
    else:
        new_request = request.override(system_message=SystemMessage(content=hint))
    return handler(new_request)
```

## 结构化输出实战

### Pydantic 模型（推荐）

```python
from pydantic import BaseModel, Field
from langchain.agents.structured_output import AutoStrategy

class BookReview(BaseModel):
    title: str = Field(description="书名")
    rating: int = Field(description="评分 1-5")
    summary: str = Field(description="50字以内的摘要")
    pros: list[str] = Field(description="优点列表")
    cons: list[str] = Field(description="缺点列表")

agent = create_agent(model, response_format=AutoStrategy(BookReview))
result = agent.invoke({"messages": [("user", "评价《三体》这本书")]})
review = result["structured_response"]
print(f"{review.title}: {review.rating}/5")
```

### TypedDict

```python
from typing import TypedDict
from langchain.agents.structured_output import ProviderStrategy

class SentimentResult(TypedDict):
    text: str
    sentiment: str  # "positive", "negative", "neutral"
    confidence: float

agent = create_agent(model, response_format=ProviderStrategy(SentimentResult))
```

### JSON Schema

```python
from langchain.agents.structured_output import ToolStrategy

weather_schema = {
    "type": "object",
    "properties": {
        "city": {"type": "string", "description": "城市名"},
        "temperature": {"type": "number", "description": "温度（摄氏度）"},
        "condition": {"type": "string", "description": "天气状况"},
    },
    "required": ["city", "temperature", "condition"],
}

agent = create_agent(model, response_format=ToolStrategy(weather_schema))
```

## 嵌入模型实战

```python
from langchain.embeddings import init_embeddings

embeddings = init_embeddings("openai:text-embedding-3-small")

# 嵌入查询文本
query_vector = embeddings.embed_query("什么是机器学习？")
print(f"向量维度：{len(query_vector)}")

# 嵌入文档
doc_vectors = embeddings.embed_documents([
    "Python 是一种编程语言",
    "机器学习是人工智能的子领域",
])
print(f"嵌入了 {len(doc_vectors)} 个文档")

# 异步版本
query_vector = await embeddings.aembed_query("什么是深度学习？")
```

## 速率限制实战

```python
from langchain.chat_models import init_chat_model
from langchain.rate_limiters import InMemoryRateLimiter

# 创建速率限制器
limiter = InMemoryRateLimiter(
    requests_per_second=5,   # 每秒 5 个请求
    max_bucket_size=10,      # 允许短时突发到 10 个
)

# 传给模型
model = init_chat_model(
    "anthropic:claude-sonnet-4-20250514",
    model_provider="anthropic",
    rate_limiter=limiter,
)

# 所有请求自动受速率限制
for i in range(20):
    result = model.invoke(f"第 {i} 个问题")
```

## 完整示例：客服智能体

将所有 API 组合起来，构建一个生产级的客服智能体：

```python
import os

from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain.agents.middleware import (
    HumanInTheLoopMiddleware,
    ModelRetryMiddleware,
    ModelCallLimitMiddleware,
    PIIMiddleware,
    ContextEditingMiddleware,
    ClearToolUsesEdit,
    SummarizationMiddleware,
)
from langchain.agents.structured_output import AutoStrategy
from langchain.rate_limiters import InMemoryRateLimiter
from langchain_core.tools import tool
from langgraph.checkpoint.memory import MemorySaver
from pydantic import BaseModel, Field

# ---- 工具定义 ----

@tool
def query_order(order_id: str) -> str:
    """查询订单状态。"""
    return f"订单 {order_id}：已发货，预计明天送达"

@tool
def create_return(order_id: str, reason: str) -> str:
    """创建退货申请。"""
    return f"已为订单 {order_id} 创建退货申请"

# ---- 速率限制 ----

limiter = InMemoryRateLimiter(requests_per_second=5)

# ---- 模型 ----

model = init_chat_model(
    "anthropic:claude-sonnet-4-20250514",
    model_provider="anthropic",
    rate_limiter=limiter,
)

# ---- 智能体 ----

agent = create_agent(
    model,
    tools=[query_order, create_return],
    system_prompt="你是一个专业的客服助手。礼貌、准确、及时地回答客户问题。",
    middleware=[
        # 退货需人工确认
        HumanInTheLoopMiddleware(
            interrupt_on={"create_return": {"allowed_decisions": ["approve", "reject"]}},
        ),
        # 模型调用失败重试
        ModelRetryMiddleware(max_retries=2),
        # 限制调用次数
        ModelCallLimitMiddleware(run_limit=20),
        # 保护客户隐私
        PIIMiddleware("email", strategy="mask"),
        PIIMiddleware("phone", strategy="redact"),
        # 压缩长对话
        ContextEditingMiddleware(
            edits=[ClearToolUsesEdit(trigger=80_000, keep=3)],
        ),
    ],
    checkpointer=MemorySaver(),  # 对话持久化
)

# ---- 使用 ----

config = {"configurable": {"thread_id": "customer-456"}}
result = agent.invoke(
    {"messages": [("user", "我的订单 ORD-123 到哪了？")]},
    config=config,
)
```

## 小结

| API | 用途 | 关键参数 |
|-----|------|---------|
| `init_chat_model()` | 初始化聊天模型 | `model`, `model_provider`, `rate_limiter` |
| `create_agent()` | 创建智能体 | `model`, `tools`, `middleware`, `response_format`, `checkpointer` |
| `@tool` | 定义工具 | docstring, 参数类型 |
| `HumanInTheLoopMiddleware` | 人工审批 | `interrupt_on` |
| `ModelRetryMiddleware` | 模型重试 | `max_retries`, `retry_on` |
| `ModelFallbackMiddleware` | 模型回退 | 位置参数：备用模型列表 |
| `PIIMiddleware` | PII 保护 | `pii_type`, `strategy` |
| `AutoStrategy` | 自动结构化输出 | `schema` |
| `init_embeddings()` | 初始化嵌入模型 | `model`, `provider` |
| `InMemoryRateLimiter` | 速率限制 | `requests_per_second`, `max_bucket_size` |