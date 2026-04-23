# 第04章：智能体

**学习目标：** 深入理解 `create_agent()` 和 `AgentState`，掌握智能体的完整生命周期。

## create_agent()

`create_agent()` 是 LangChain v1 的核心函数，用于创建智能体。

### 完整签名

```python
def create_agent(
    model: str | BaseChatModel,
    tools: Sequence[BaseTool | Callable | dict] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware] = (),
    response_format: ResponseFormat | type | dict | None = None,
    state_schema: type[AgentState] | None = None,
    context_schema: type[ContextT] | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    interrupt_before: list[str] | None = None,
    interrupt_after: list[str] | None = None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache | None = None,
) -> CompiledStateGraph
```

### 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| `model` | `str \| BaseChatModel` | 模型名称字符串或模型实例 |
| `tools` | `Sequence \| None` | 可用工具列表 |
| `system_prompt` | `str \| SystemMessage \| None` | 系统提示词 |
| `middleware` | `Sequence[AgentMiddleware]` | 中间件列表 |
| `response_format` | `ResponseFormat \| None` | 结构化输出格式 |
| `state_schema` | `type[AgentState] \| None` | 自定义状态 schema |
| `context_schema` | `type \| None` | 运行时上下文 schema |
| `checkpointer` | `Checkpointer \| None` | 状态持久化检查点 |
| `store` | `BaseStore \| None` | 跨线程持久存储 |
| `interrupt_before` | `list[str] \| None` | 在指定节点前中断 |
| `interrupt_after` | `list[str] \| None` | 在指定节点后中断 |
| `debug` | `bool` | 启用调试输出 |
| `name` | `str \| None` | 智能体名称 |
| `cache` | `BaseCache \| None` | LangGraph 缓存 |

### 基本用法

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

# 最简单的智能体
model = init_chat_model("claude-sonnet-4-20250514", model_provider="anthropic")
agent = create_agent(model)

# 带工具和系统提示
agent = create_agent(
    model,
    tools=[search_tool, calculate_tool],
    system_prompt="你是一个数据分析助手，善于使用工具来回答问题。",
)
```

### 用模型字符串创建

`model` 参数接受字符串，格式为 `"提供商:模型"`：

```python
# 等价于先 init_chat_model 再 create_agent
agent = create_agent("anthropic:claude-sonnet-4-20250514")
```

## AgentState

`AgentState` 是智能体的状态类型，定义了智能体运行时的数据结构。

### 默认结构

```python
class AgentState(TypedDict, Generic[ResponseT]):
    messages: Required[Annotated[list[AnyMessage], add_messages]]
    jump_to: NotRequired[Annotated[JumpTo | None, EphemeralValue, PrivateStateAttr]]
    structured_response: NotRequired[Annotated[ResponseT, OmitFromInput]]
```

三个字段：

| 字段 | 说明 |
|------|------|
| `messages` | 对话消息列表，自动合并新旧消息 |
| `jump_to` | 控制流跳转目标：`"tools"`、`"model"` 或 `"end"` |
| `structured_response` | 结构化输出结果，不出现在输入中 |

### 自定义状态字段

通过继承 `AgentState` 添加自定义字段：

```python
from typing import TypedDict, Annotated
from typing_extensions import NotRequired
from langchain.agents import AgentState

class MyState(AgentState):
    # 添加自定义状态
    user_name: NotRequired[str]
    search_history: NotRequired[list[str]]
```

然后传给 `create_agent()`：

```python
agent = create_agent(model, state_schema=MyState)
```

### OmitFromSchema

`OmitFromSchema` 控制字段是否出现在输入/输出 schema 中：

```python
from langchain.agents.middleware.types import OmitFromSchema, OmitFromInput, OmitFromOutput

class MyState(AgentState):
    # 内部状态，不出现在输入和输出 schema 中
    internal_count: NotRequired[Annotated[int, PrivateStateAttr]]
```

- `OmitFromInput`：字段不出现在输入 schema 中
- `OmitFromOutput`：字段不出现在输出 schema 中
- `PrivateStateAttr`：等同于 `OmitFromInput + OmitFromOutput`

## 智能体执行循环

智能体的核心是一个循环：模型调用 → 工具调用 → 模型调用 → ...

```
用户输入
   ↓
模型生成回复
   ↓
回复包含工具调用？──否──→ 返回结果
   ↓ 是
执行工具
   ↓
工具结果返回给模型
   ↓
模型继续生成（回到"回复包含工具调用？"）
```

这个循环由 LangGraph 在内部管理，你通常不需要手动控制。

## 控制流

### JumpTo — 跳转控制

在中间件中，可以通过设置 `jump_to` 控制智能体的下一步：

```python
from langchain.agents.middleware.types import JumpTo

# JumpTo 可以是以下三个值：
# "tools"  → 跳到工具执行节点
# "model"  → 跳回模型调用节点
# "end"    → 结束智能体运行
```

### Command — 高级控制

`Command` 提供更精细的控制能力：

```python
from langgraph.types import Command

# 更新状态并指定下一步
cmd = Command(update={"messages": [...]}, goto="model")

# 向多个节点并行发送
cmd = Command(send=[...])
```

### Send — 并行分发

`Send` 用于并行执行多个任务：

```python
from langgraph.types import Send

# 并行调用多个工具
sends = [Send("tools", {"tool_call": call}) for call in tool_calls]
```

## Runtime 和状态持久化

`Runtime` 提供对 LangGraph 运行时的访问：

```python
from langgraph.runtime import Runtime

# 在中间件中访问运行时
def my_hook(state, runtime: Runtime):
    thread_id = runtime.config["configurable"]["thread_id"]
```

### 持久化

使用 `checkpointer` 保存状态，让智能体可以恢复对话：

```python
from langgraph.checkpoint.memory import MemorySaver

agent = create_agent(
    model,
    checkpointer=MemorySaver(),  # 内存中的检查点
)

# 同一 thread_id 的对话会自动恢复上下文
config = {"configurable": {"thread_id": "user-123"}}
result1 = agent.invoke({"messages": [("user", "我叫小明")]}, config=config)
result2 = agent.invoke({"messages": [("user", "我叫什么名字？")]}, config=config)
# 智能体会记住"小明"
```

## 编译图与 LangGraph

`create_agent()` 返回一个 `CompiledStateGraph`——这是 LangGraph 编译后的状态图。LangChain v1 的智能体构建在 LangGraph 之上：

- **LangChain v1**：高层 API，快速创建智能体
- **LangGraph**：底层框架，精细控制工作流

当你需要 LangChain v1 无法满足的自定义逻辑时，可以直接使用 LangGraph。参见 [LangGraph 文档](https://docs.langchain.com/oss/python/langgraph/overview)。

## 下一步

- [第05章：中间件概览](05-middleware-overview.md) — 用中间件增强智能体
- [第07章：结构化输出](07-structured-output.md) — 让智能体返回结构化数据

## 常见问题

### 智能体运行时间过长

**原因：** 工具调用循环没有终止条件，模型持续调用工具。

**解决：** 使用 `ModelCallLimitMiddleware` 或 `ToolCallLimitMiddleware` 限制调用次数。参见[第06章](06-built-in-middleware.md)。

### 状态丢失，对话没有记忆

**原因：** 没有配置 `checkpointer`。

**解决：** 在 `create_agent()` 中传入 `checkpointer=MemorySaver()`，并在调用时指定 `thread_id`。

### 自定义状态字段在输出中不可见

**原因：** 字段被标记为 `PrivateStateAttr` 或 `OmitFromOutput`。

**解决：** 如果需要在输出中看到该字段，移除 `PrivateStateAttr` 标注，改为仅用 `OmitFromInput`。