# 第05章：中间件概览

**学习目标：** 理解中间件的核心理念和装饰器 API，能够编写自定义中间件。

## 什么是中间件？

中间件是拦截和修改智能体行为的插件。它可以在不改变核心逻辑的情况下，为智能体添加功能：重试失败调用、检测敏感信息、限制调用次数等。

中间件的执行类似洋葱模型——外层中间件包裹内层：

```
请求 → 中间件A(外层) → 中间件B(内层) → 模型/工具
                  ←          ←         ←
响应 ← 中间件A(外层) ← 中间件B(内层) ←
```

## AgentMiddleware 协议

所有中间件都实现 `AgentMiddleware` 协议。你可以通过类继承或装饰器来创建中间件。

### 类方式

```python
from langchain.agents.middleware import AgentMiddleware

class MyMiddleware(AgentMiddleware):
    def before_model(self, state, runtime):
        # 在模型调用前执行
        return None  # 返回 None 表示不修改状态

    def after_model(self, state, runtime):
        # 在模型调用后执行
        return None
```

### 可选方法

`AgentMiddleware` 有以下可选方法，全部默认为空操作：

| 方法 | 触发时机 | 返回值 |
|------|---------|--------|
| `before_model(state, runtime)` | 模型调用前 | `dict \| None` |
| `after_model(state, runtime)` | 模型调用后 | `dict \| None` |
| `before_agent(state, runtime)` | 智能体步骤前 | `dict \| None` |
| `after_agent(state, runtime)` | 智能体步骤后 | `dict \| None` |
| `wrap_model_call(request, handler)` | 包装模型调用 | `ModelResponse` |
| `wrap_tool_call(request, handler)` | 包装工具调用 | `ToolMessage` |

每个方法都有对应的异步版本（加 `a` 前缀，如 `abefore_model`）。

## 六个装饰器钩子

LangChain v1 提供六个装饰器，让你快速定义中间件：

### before_model — 模型调用前

```python
from langchain.agents.middleware import before_model

@before_model
def log_request(state, runtime):
    """在每次模型调用前记录消息数量。"""
    print(f"即将调用模型，当前消息数：{len(state['messages'])}")
```

### after_model — 模型调用后

```python
from langchain.agents.middleware import after_model

@after_model
def log_response(state, runtime):
    """在模型返回后记录回复长度。"""
    last_msg = state["messages"][-1]
    print(f"模型回复长度：{len(last_msg.content)}")
```

### before_agent — 智能体步骤前

```python
from langchain.agents.middleware import before_agent

@before_agent
def step_counter(state, runtime):
    """在每步前记录步骤数。"""
    print("智能体开始新一步")
```

### after_agent — 智能体步骤后

```python
from langchain.agents.middleware import after_agent

@after_agent
def step_done(state, runtime):
    """在每步后确认完成。"""
    print("智能体完成一步")
```

### wrap_model_call — 包装模型调用

`wrap_model_call` 是最强大的钩子，可以完全控制模型调用过程。它接收 `request` 和 `handler`：

```python
from langchain.agents.middleware import wrap_model_call

@wrap_model_call
def log_model_call(request, handler):
    """记录模型调用的输入输出。"""
    print(f"调用模型：{len(request.messages)} 条消息")
    response = handler(request)
    print(f"模型返回：{len(response.result)} 条消息")
    return response
```

`handler(request)` 执行实际的模型调用（包括内层中间件）。你可以在调用前后添加逻辑，甚至完全替换调用。

### wrap_tool_call — 包装工具调用

```python
from langchain.agents.middleware import wrap_tool_call

@wrap_tool_call
def log_tool_call(request, handler):
    """记录工具调用。"""
    print(f"调用工具：{request.tool_call['name']}")
    result = handler(request)
    print(f"工具返回：{result.content[:50]}")
    return result
```

## 装饰器的共享参数

四个 `before/after` 装饰器共享以下关键字参数：

```python
@before_model(
    state_schema=MyState,    # 指定状态类型
    tools=[my_tool],          # 中间件注册额外工具
    can_jump_to=["end"],      # 允许跳转的目标
    name="my_hook",           # 中间件名称
)
def my_hook(state, runtime):
    ...
```

## ModelRequest 和 ModelResponse

### ModelRequest

`wrap_model_call` 的 `request` 参数包含模型调用的所有信息：

```python
request.model           # BaseChatModel 实例
request.messages        # list[AnyMessage] 消息列表
request.system_message  # SystemMessage | None 系统消息
request.tool_choice     # Any | None 工具选择策略
request.tools           # list[BaseTool] 可用工具
request.response_format # ResponseFormat | None 结构化输出格式
request.state           # AgentState 当前状态
request.runtime         # Runtime 运行时
request.model_settings  # dict 模型额外设置
```

### override() 方法

`ModelRequest` 提供 `override()` 方法创建修改后的副本（不可变设计）：

```python
# 修改系统消息
new_request = request.override(system_message=SystemMessage(content="新指令"))

# 替换模型
new_request = request.override(model=another_model)

# 修改消息
new_request = request.override(messages=[...])
```

### ModelResponse

`handler(request)` 返回 `ModelResponse`：

```python
response.result               # list[BaseMessage] 模型返回的消息
response.structured_response  # ResponseT | None 结构化输出
```

### ExtendedModelResponse

当需要返回 `Command`（如跳转或并行分发）时，使用 `ExtendedModelResponse`：

```python
from langchain.agents.middleware import ExtendedModelResponse

# 在 wrap_model_call 中返回扩展响应
return ExtendedModelResponse(
    model_response=response,
    command=Command(goto="end"),
)
```

## dynamic_prompt — 动态修改系统提示

`dynamic_prompt` 专用于根据运行时状态动态生成系统提示：

```python
from langchain.agents.middleware import dynamic_prompt

@dynamic_prompt
def personalized_prompt(request):
    """根据用户信息生成个性化系统提示。"""
    user_name = request.state.get("user_name", "用户")
    return f"你是一个专属助手，正在为 {user_name} 服务。"
```

## hook_config — 配置钩子行为

`hook_config` 用于设置中间件方法的元数据，如允许的跳转目标：

```python
from langchain.agents.middleware import hook_config, before_model

@hook_config(can_jump_to=["end", "tools"])
@before_model
def conditional_stop(state, runtime):
    """在特定条件下允许跳转到工具节点或结束。"""
    if should_stop(state):
        return Command(goto="end")
```

## 编写自定义中间件

完整示例：一个限制回复长度的中间件：

```python
from langchain.agents.middleware import wrap_model_call, AgentMiddleware
from langchain.agents.middleware.types import ModelRequest, ModelResponse

class LengthLimitMiddleware(AgentMiddleware):
    """限制模型回复的长度。"""

    def __init__(self, max_length: int = 500):
        self.max_length = max_length

    def wrap_model_call(self, request, handler):
        # 在系统消息中添加长度限制指令
        limit_msg = f"请将回复控制在 {self.max_length} 字以内。"
        if request.system_message:
            new_system = request.system_message.content + "\n" + limit_msg
            new_request = request.override(
                system_message=SystemMessage(content=new_system)
            )
        else:
            new_request = request.override(
                system_message=SystemMessage(content=limit_msg)
            )
        return handler(new_request)
```

## 使用中间件

将中间件传给 `create_agent()`：

```python
from langchain.agents import create_agent

agent = create_agent(
    model,
    tools=[search_tool],
    middleware=[
        LengthLimitMiddleware(max_length=200),
        # 可以组合多个中间件
    ],
)
```

## 下一步

- [第06章：内置中间件](06-built-in-middleware.md) — 了解 14 个内置中间件
- [第07章：结构化输出](07-structured-output.md) — 结构化输出与中间件

## 常见问题

### 中间件没有执行

**原因：** 中间件没有被传递给 `create_agent()` 的 `middleware` 参数。

**解决：** 确保中间件实例在 `middleware` 列表中。

### wrap_model_call 中 override 不生效

**原因：** `override()` 返回新对象，不修改原对象。如果你忘记使用返回值，修改不会生效。

**解决：** 始终使用 `new_request = request.override(...)` 并将 `new_request` 传给 `handler`。

### 多个中间件的执行顺序

**原因：** 中间件按列表顺序包裹，外层先执行 `before`，内层先执行 `after`。

**解决：** 将需要最先/最后执行的中间件放在列表前面。理解洋葱模型：`middleware[0]` 是最外层。