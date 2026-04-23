# 第07章：结构化输出

**学习目标：** 让智能体返回结构化数据而非自由文本。

## 概述

默认情况下，智能体返回自由文本回复。但很多场景需要结构化数据——例如表单填写、数据提取、分类结果。LangChain v1 通过 `response_format` 参数实现结构化输出。

## ResponseFormat

`ResponseFormat` 支持四种 schema 类型：

```python
from pydantic import BaseModel, Field
from typing import TypedDict
from dataclasses import dataclass

# 方式一：Pydantic 模型（推荐）
class UserInfo(BaseModel):
    name: str = Field(description="用户姓名")
    age: int = Field(description="用户年龄")
    email: str = Field(description="用户邮箱")

# 方式二：dataclass
@dataclass
class ProductInfo:
    name: str
    price: float
    in_stock: bool

# 方式三：TypedDict
class SearchResult(TypedDict):
    title: str
    url: str
    relevance: float

# 方式四：JSON Schema 字典
weather_schema = {
    "type": "object",
    "properties": {
        "temperature": {"type": "number", "description": "温度（摄氏度）"},
        "condition": {"type": "string", "description": "天气状况"},
    },
    "required": ["temperature", "condition"],
}
```

## 三种策略

### AutoStrategy — 自动选择

`AutoStrategy` 根据模型能力自动选择最佳方式：

```python
from langchain.agents.structured_output import AutoStrategy

agent = create_agent(
    model,
    response_format=AutoStrategy(UserInfo),
)
```

如果模型支持原生结构化输出，使用 `ProviderStrategy`；否则回退到 `ToolStrategy`。

### ProviderStrategy — 使用提供商原生能力

直接使用模型提供商的结构化输出功能（如 OpenAI 的 structured outputs）：

```python
from langchain.agents.structured_output import ProviderStrategy

agent = create_agent(
    model,
    response_format=ProviderStrategy(UserInfo, strict=True),
)
```

`strict=True` 启用严格模式，确保输出完全符合 schema。

### ToolStrategy — 使用工具调用模拟

通过添加一个"输出工具"来获取结构化数据。兼容所有支持工具调用的模型：

```python
from langchain.agents.structured_output import ToolStrategy

agent = create_agent(
    model,
    response_format=ToolStrategy(
        UserInfo,
        tool_message_content="用户信息已提取",
        handle_errors=True,  # 验证失败时返回错误给模型重试
    ),
)
```

`handle_errors` 支持以下值：
- `True`（默认）：返回错误信息让模型重试
- `False`：直接抛出异常
- 异常类型或元组：仅处理指定异常
- 函数：`Callable[[Exception], str]` 自定义错误消息

## 简写形式

`create_agent()` 的 `response_format` 参数接受多种类型：

```python
# 直接传 schema 类型——等价于 AutoStrategy
agent = create_agent(model, response_format=UserInfo)

# 传字典——作为 JSON Schema
agent = create_agent(model, response_format=weather_schema)

# 显式使用策略
agent = create_agent(model, response_format=AutoStrategy(UserInfo))
agent = create_agent(model, response_format=ProviderStrategy(UserInfo))
agent = create_agent(model, response_format=ToolStrategy(UserInfo))
```

## 获取结构化输出

智能体运行后，结构化输出在 `structured_response` 字段中：

```python
result = agent.invoke({"messages": [("user", "我叫张三，25岁，邮箱 zhangsan@example.com")]})

# 获取结构化输出
user_info = result["structured_response"]
print(user_info.name)   # "张三"
print(user_info.age)     # 25
print(user_info.email)   # "zhangsan@example.com"
```

## 错误处理

### StructuredOutputError

所有结构化输出错误的基类：

```python
from langchain.agents.structured_output import StructuredOutputError

try:
    result = agent.invoke(...)
except StructuredOutputError as e:
    print(f"结构化输出失败：{e}")
    print(f"AI 原始消息：{e.ai_message}")
```

### MultipleStructuredOutputsError

模型返回了多个结构化输出工具调用（只允许一个）：

```python
from langchain.agents.structured_output import MultipleStructuredOutputsError

try:
    result = agent.invoke(...)
except MultipleStructuredOutputsError as e:
    print(f"模型错误地返回了多个结构化输出：{e.tool_names}")
```

### StructuredOutputValidationError

结构化输出的内容验证失败：

```python
from langchain.agents.structured_output import StructuredOutputValidationError

try:
    result = agent.invoke(...)
except StructuredOutputValidationError as e:
    print(f"验证失败，工具名：{e.tool_name}")
    print(f"原始错误：{e.source}")
```

## ProviderStrategyBinding 和 OutputToolBinding

这些是内部绑定类型，将 schema 与具体的输出策略关联：

- `ProviderStrategyBinding`：`ProviderStrategy` 的绑定结果，提供 `parse(response)` 方法
- `OutputToolBinding`：`ToolStrategy` 的绑定结果，提供 `parse(tool_args)` 方法

通常不需要直接使用，`create_agent()` 内部自动处理。

## 与中间件配合

结构化输出可以与中间件组合使用：

```python
agent = create_agent(
    model,
    tools=[search_tool],
    response_format=AutoStrategy(UserInfo),
    middleware=[
        ModelRetryMiddleware(max_retries=2),
        PIIMiddleware("email", strategy="mask"),
    ],
)
```

## 下一步

- [第06章：内置中间件](06-built-in-middleware.md) — 了解更多中间件
- [第04章：智能体](04-agents.md) — 智能体状态与 AgentState

## 常见问题

### 结构化输出为 None

**原因：** 模型没有生成符合 schema 的输出，或策略不匹配模型能力。

**解决：** 尝试使用 `ToolStrategy`（兼容性最好），或在 system_prompt 中明确要求输出特定格式。

### Pydantic 验证失败

**原因：** 模型生成的字段值不符合 schema 定义的类型。

**解决：** 使用 `Field(description=...)` 和 `Field(examples=...)` 为模型提供更清晰的指引，或使用 `ToolStrategy(handle_errors=True)` 让模型重试。

### AutoStrategy 选错了策略

**原因：** 自动判断可能不总是最优。

**解决：** 明确指定策略。如果模型支持原生结构化输出，用 `ProviderStrategy`；否则用 `ToolStrategy`。