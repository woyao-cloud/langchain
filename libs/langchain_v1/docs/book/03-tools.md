# 第03章：工具

**学习目标：** 定义和注册工具，让智能体与外部世界交互。

## 什么是工具？

工具是智能体可以调用的函数。当模型判断需要执行某个操作（如搜索、计算、读写文件）时，它会发起工具调用，智能体执行工具并将结果返回给模型。

## 用 @tool 装饰器定义工具

最常用的方式是使用 `@tool` 装饰器：

```python
from langchain_core.tools import tool

@tool
def search_weather(city: str) -> str:
    """查询指定城市的天气信息。

    Args:
        city: 城市名称，如"北京"、"上海"
    """
    # 实际场景中这里会调用天气 API
    return f"{city}：晴天，气温 25°C"
```

!!! warning
    工具的 docstring 非常重要——模型根据它来判断何时调用工具。务必清晰描述工具的用途和参数含义。

### 带默认参数的工具

```python
@tool
def search_database(
    query: str,
    limit: int = 10,
    offset: int = 0,
) -> list[dict]:
    """搜索数据库记录。

    Args:
        query: 搜索关键词
        limit: 返回结果数量上限，默认 10
        offset: 结果偏移量，默认 0
    """
    return [{"id": i, "query": query} for i in range(offset, offset + limit)]
```

## 用 StructuredTool 定义工具

当需要更灵活的控制时，使用 `StructuredTool` 类：

```python
from langchain_core.tools import StructuredTool
from pydantic import BaseModel, Field

# 定义参数 schema
class SearchInput(BaseModel):
    query: str = Field(description="搜索关键词")
    max_results: int = Field(default=5, description="最大返回数量")

# 定义工具函数
def search_func(query: str, max_results: int = 5) -> list[str]:
    return [f"结果{i}: {query}" for i in range(max_results)]

# 创建工具
search_tool = StructuredTool.from_function(
    func=search_func,
    name="search",
    description="搜索相关内容",
    args_schema=SearchInput,
)
```

## 异步工具

工具可以同时提供同步和异步实现：

```python
@tool
async def async_search(query: str) -> str:
    """异步搜索内容。"""
    # 异步操作，如调用远程 API
    return f"搜索结果：{query}"

# 或使用 StructuredTool
search_tool = StructuredTool.from_function(
    func=sync_search,        # 同步版本
    coroutine=async_search,  # 异步版本
    name="search",
    description="搜索相关内容",
)
```

## ToolNode

`ToolNode` 是 LangGraph 中执行工具的节点。`create_agent()` 内部自动使用它，但你也可以直接使用：

```python
from langchain.tools import ToolNode

# 创建工具节点
tool_node = ToolNode([search_weather, search_database])

# 工具节点接收包含 tool_calls 的 AIMessage，返回 ToolMessage
```

## 注入：InjectedState、InjectedStore、ToolRuntime

工具可以通过注入访问智能体的运行时状态：

### InjectedState — 访问智能体状态

```python
from langchain.tools import InjectedState
from langchain_core.tools import tool
from typing import Annotated

@tool
def summarize_conversation(
    state: Annotated[dict, InjectedState],
) -> str:
    """总结当前对话内容。"""
    messages = state["messages"]
    return f"对话共 {len(messages)} 条消息"
```

### InjectedStore — 访问持久存储

```python
from langchain.tools import InjectedStore

@tool
def save_note(
    content: str,
    store: Annotated[object, InjectedStore],
) -> str:
    """保存笔记到持久存储。"""
    # 使用 store 进行持久化操作
    return f"已保存笔记：{content[:20]}..."
```

### ToolRuntime — 访问运行时信息

```python
from langchain.tools import ToolRuntime

@tool
def get_runtime_info(
    runtime: Annotated[object, ToolRuntime],
) -> str:
    """获取当前运行时信息。"""
    return f"运行时 ID：{runtime.run_id}"
```

## ToolCallRequest 和 ToolCallWithContext

这些类型用于中间件中拦截和修改工具调用：

```python
from langchain.tools import ToolCallRequest, ToolCallWithContext

# ToolCallRequest：包含工具名称、参数和调用 ID
# ToolCallWithContext：扩展 ToolCallRequest，附加上下文信息
```

详细用法参见[第05章：中间件概览](05-middleware-overview.md)。

## 工具的错误处理

工具执行出错时，返回的错误信息会传回模型，让它决定下一步操作：

```python
@tool
def risky_operation(url: str) -> str:
    """访问指定 URL 并获取内容。"""
    try:
        # 可能失败的操作
        return fetch_url(url)
    except ConnectionError as e:
        # 返回错误信息给模型
        return f"访问失败：{e}"
```

!!! tip
    不要在工具中抛出未捕获的异常。将错误信息作为字符串返回，让模型自行决定是否重试或换一种方式。

## 与智能体集成

将工具传递给 `create_agent()`：

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

model = init_chat_model("claude-sonnet-4-20250514", model_provider="anthropic")

agent = create_agent(
    model,
    tools=[search_weather, summarize_conversation],
)

result = agent.invoke({"messages": [("user", "北京今天天气怎么样？")]})
```

## 下一步

- [第04章：智能体](04-agents.md) — 深入理解智能体的工作机制
- [第05章：中间件概览](05-middleware-overview.md) — 用中间件增强工具行为

## 常见问题

### 工具没有被调用

**原因：** 工具的 docstring 描述不够清晰，模型无法判断何时使用。

**解决：** 改进 docstring，明确说明工具的使用场景。例如，将"搜索"改为"当需要搜索互联网上的实时信息时使用此工具"。

### 工具参数类型错误

**原因：** 模型生成的参数类型与工具定义不匹配。

**解决：** 使用 `Field(description=...)` 为每个参数添加清晰的描述和示例，帮助模型生成正确类型的参数。

### InjectedState 获取不到状态

**原因：** `InjectedState` 注入需要 `Annotated` 类型标注格式正确。

**解决：** 确保格式为 `Annotated[dict, InjectedState]`，第一个类型参数是状态字典类型。