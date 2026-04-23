# LangChain v1 中文文档书 实现计划

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 为 langchain_v1 包编写 10 章中文文档书，覆盖全部公共 API，面向初学者。

**Architecture:** 10 个 Markdown 文件存放在 `libs/langchain_v1/docs/book/`，按概念递进组织。每章包含概念说明、可运行代码示例、教程演练和常见问题。第10章为 API 索引页。

**Tech Stack:** Markdown、Python 3.10+、langchain v1.2.15

---

## 文件结构

| 文件 | 职责 |
|------|------|
| `docs/book/01-getting-started.md` | 快速开始：安装、第一个智能体 |
| `docs/book/02-messages-and-models.md` | 消息类型体系、init_chat_model() |
| `docs/book/03-tools.md` | 工具定义、ToolNode、注入 |
| `docs/book/04-agents.md` | create_agent()、AgentState、执行循环 |
| `docs/book/05-middleware-overview.md` | 中间件理念、装饰器 API |
| `docs/book/06-built-in-middleware.md` | 14 个内置中间件详解 |
| `docs/book/07-structured-output.md` | 结构化输出策略 |
| `docs/book/08-embeddings.md` | init_embeddings() |
| `docs/book/09-rate-limiters.md` | 速率限制器 |
| `docs/book/10-api-reference.md` | 公共 API 索引 |

所有文件路径基于 `libs/langchain_v1/`。

---

### Task 1: 创建目录结构和第01章

**Files:**
- Create: `docs/book/01-getting-started.md`

- [ ] **Step 1: 创建 book 目录**

```bash
mkdir -p libs/langchain_v1/docs/book
```

- [ ] **Step 2: 编写第01章**

写入 `docs/book/01-getting-started.md`：

```markdown
# 第01章：快速开始

**学习目标：** 5分钟内创建并运行你的第一个 LangChain v1 智能体。

## 安装

LangChain v1 是一个 Python 包，通过 pip 安装：

```bash
# 安装核心包
pip install langchain
```

LangChain 本身不包含任何模型提供商。你需要额外安装至少一个提供商包：

```bash
# Anthropic（Claude 系列）
pip install langchain[anthropic]

# OpenAI（GPT 系列）
pip install langchain[openai]

# Ollama（本地模型）
pip install langchain[ollama]
```

完整的提供商列表参见[第02章](02-messages-and-models.md)。

## 配置 API 密钥

每个提供商需要对应的 API 密钥。通过环境变量设置：

```bash
# Anthropic
export ANTHROPIC_API_KEY="your-key-here"

# OpenAI
export OPENAI_API_KEY="your-key-here"
```

或者在代码中设置（适合本地实验）：

```python
import os

os.environ["ANTHROPIC_API_KEY"] = "your-key-here"
```

!!! warning
    不要将 API 密钥硬编码到代码中并提交到版本控制。生产环境应使用 `.env` 文件或密钥管理服务。

## 第一个智能体

下面是创建一个 LangChain v1 智能体的最简示例：

```python
# 导入核心模块
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

# 初始化聊天模型
# init_chat_model() 会根据模型名称自动选择提供商
model = init_chat_model("claude-sonnet-4-20250514", model_provider="anthropic")

# 创建智能体
agent = create_agent(model)

# 运行智能体
result = agent.invoke({"messages": [("user", "你好，请介绍一下你自己")]})
print(result)
```

## 添加工具

智能体的真正威力在于工具——让它与外部世界交互：

```python
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model
from langchain_core.tools import tool

# 定义一个简单工具
@tool
def add_numbers(a: int, b: int) -> int:
    """将两个数字相加并返回结果。"""
    return a + b

# 创建带工具的智能体
model = init_chat_model("claude-sonnet-4-20250514", model_provider="anthropic")
agent = create_agent(model, tools=[add_numbers])

# 运行——智能体会自动判断何时调用工具
result = agent.invoke({"messages": [("user", "3加5等于几？")]})
print(result)
```

## 解读输出

智能体的输出是一个字典，包含以下关键字段：

```python
result = agent.invoke({"messages": [("user", "你好")]})

# result 包含：
# - messages: 对话消息列表（包括用户消息、AI 回复和工具调用结果）
# - structured_response: 结构化输出（如果配置了 response_format）
```

`messages` 列表中的每个元素都是一种消息类型：
- `HumanMessage`：用户发送的消息
- `AIMessage`：AI 的回复（可能包含工具调用）
- `ToolMessage`：工具执行的结果

## 下一步

现在你已经运行了第一个智能体！接下来的章节会逐步深入：

- [第02章：消息与模型](02-messages-and-models.md) — 理解消息系统和聊天模型
- [第03章：工具](03-tools.md) — 深入学习工具的定义和使用
- [第04章：智能体](04-agents.md) — 掌握智能体的完整能力

## 常见问题

### 导入错误：`ModuleNotFoundError: No module named 'langchain_anthropic'`

**原因：** 没有安装对应的提供商包。

**解决：** 运行 `pip install langchain[anthropic]` 或对应的提供商包。

### API 密钥错误：`AuthenticationError`

**原因：** API 密钥未设置或已过期。

**解决：** 检查环境变量是否正确设置，密钥是否有效。

### 智能体没有调用工具

**原因：** 模型认为不需要调用工具就能回答问题，或者工具描述不够清晰。

**解决：** 确保工具的 docstring 清晰描述了工具的用途和使用场景，让模型能准确判断何时调用。
```

- [ ] **Step 3: 提交**

```bash
git add libs/langchain_v1/docs/book/01-getting-started.md
git commit -m "docs(langchain): add chapter 01 - getting started"
```

---

### Task 2: 第02章 消息与模型

**Files:**
- Create: `docs/book/02-messages-and-models.md`

- [ ] **Step 1: 编写第02章**

写入 `docs/book/02-messages-and-models.md`：

```markdown
# 第02章：消息与模型

**学习目标：** 理解 LangChain 的消息系统，掌握 `init_chat_model()` 的全部用法。

## 消息类型体系

LangChain 使用不同类型的消息来表示对话中的各方角色：

```python
from langchain.messages import (
    HumanMessage,    # 用户消息
    AIMessage,       # AI 回复
    SystemMessage,   # 系统指令
    ToolMessage,     # 工具执行结果
    RemoveMessage,   # 删除指定消息（用于管理上下文）
)
```

### 创建消息

```python
# 用户消息
human_msg = HumanMessage(content="你好，请帮我分析这段代码")

# AI 回复
ai_msg = AIMessage(content="好的，我来帮你分析这段代码...")

# 系统消息——设定 AI 的行为方式
system_msg = SystemMessage(content="你是一个专业的 Python 代码审查助手")

# 工具结果消息
tool_msg = ToolMessage(content="代码分析结果：...", tool_call_id="call_123")
```

### 简写形式

对于用户消息，可以用元组简写：

```python
# 这两种写法等价
messages = [HumanMessage(content="你好")]
messages = [("user", "你好")]

# 多条消息
messages = [
    ("system", "你是一个有帮助的助手"),
    ("user", "你好"),
]
```

## 内容块

消息的 `content` 不限于纯文本。LangChain 支持多种内容块类型：

```python
from langchain.messages import (
    TextContentBlock,      # 纯文本
    ImageContentBlock,     # 图片
    AudioContentBlock,     # 音频
    FileContentBlock,      # 文件
    DataContentBlock,      # 结构化数据
    ReasoningContentBlock, # 推理过程（思维链）
)
```

### 多模态消息示例

```python
# 发送文本+图片的混合消息
message = HumanMessage(content=[
    TextContentBlock(text="这张图片里有什么？"),
    ImageContentBlock(url="https://example.com/photo.jpg"),
])
```

## init_chat_model()

`init_chat_model()` 是 LangChain v1 初始化聊天模型的统一入口。它支持 25+ 个提供商。

### 基本用法

```python
from langchain.chat_models import init_chat_model

# 方式一：指定模型名称和提供商
model = init_chat_model("claude-sonnet-4-20250514", model_provider="anthropic")

# 方式二：使用 "提供商:模型" 前缀，自动识别提供商
model = init_chat_model("anthropic:claude-sonnet-4-20250514")

# 方式三：仅从模型名推断提供商（部分模型名可自动识别）
model = init_chat_model("gpt-4o")  # 自动识别为 openai
```

### 完整签名

```python
def init_chat_model(
    model: str | None = None,
    *,
    model_provider: str | None = None,
    configurable_fields: Literal["any"] | list[str] | tuple[str, ...] | None = None,
    config_prefix: str | None = None,
    **kwargs: Any,
) -> BaseChatModel | _ConfigurableModel
```

### 主要参数

| 参数 | 类型 | 说明 |
|------|------|------|
| `model` | `str \| None` | 模型名称，可选 "提供商:模型" 格式 |
| `model_provider` | `str \| None` | 显式指定提供商 |
| `configurable_fields` | `Literal["any"] \| list[str] \| None` | 运行时可配置的字段 |
| `config_prefix` | `str \| None` | 配置键的前缀 |
| `**kwargs` | `Any` | 传递给模型构造器的额外参数 |

### 内置提供商

```python
# 支持的提供商及其包名：
# "anthropic"       → langchain-anthropic
# "openai"          → langchain-openai
# "azure_openai"    → langchain-openai
# "ollama"          → langchain-ollama
# "google_genai"    → langchain-google-genai
# "google_vertexai" → langchain-google-vertexai
# "bedrock"         → langchain-aws
# "bedrock_converse"→ langchain-aws
# "groq"            → langchain-groq
# "fireworks"       → langchain-fireworks
# "deepseek"        → langchain-deepseek
# "mistralai"       → langchain-mistralai
# "huggingface"     → langchain-huggingface
# "together"        → langchain-together
# "xai"             → langchain-xai
# "perplexity"      → langchain-perplexity
# "baseten"         → langchain-baseten
# "azure_ai"        → langchain-azure-ai
```

### 可配置模型

当需要运行时切换模型时，使用 `configurable_fields`：

```python
# 创建可在运行时切换模型的配置
model = init_chat_model(
    configurable_fields=["model", "model_provider"],
)

# 运行时指定模型
result = model.invoke(
    "你好",
    config={"configurable": {"model": "gpt-4o", "model_provider": "openai"}}
)
```

## 流式输出

对于长回复，流式输出可以逐字显示结果，改善用户体验：

```python
# 同步流式输出
for chunk in model.stream("请写一首关于春天的诗"):
    print(chunk.content, end="", flush=True)

# 异步流式输出
async for chunk in model.astream("请写一首关于春天的诗"):
    print(chunk.content, end="", flush=True)
```

## Token 计量

每次模型调用都包含 token 使用信息：

```python
result = model.invoke("你好")
usage = result.response_metadata.get("usage", {})

# usage 包含：
# - input_tokens: 输入 token 数
# - output_tokens: 输出 token 数
# - total_tokens: 总 token 数
print(f"使用了 {usage.get('total_tokens', 0)} 个 token")
```

## 下一步

- [第03章：工具](03-tools.md) — 让智能体调用外部工具
- [第08章：嵌入模型](08-embeddings.md) — 文本向量化

## 常见问题

### `ModuleNotFoundError: No module named 'langchain_xxx'`

**原因：** 对应的提供商包未安装。

**解决：** `init_chat_model()` 会尝试自动导入提供商包。如果失败，安装对应的包：`pip install langchain-<provider>`。

### 模型名称无法自动推断提供商

**原因：** 某些模型名称在不同提供商间重复（如 `llama3`），或名称格式不被识别。

**解决：** 显式指定 `model_provider` 参数，或使用 "提供商:模型" 格式：`init_chat_model("ollama:llama3")`。

### 流式输出时获取完整内容

**原因：** 流式输出的每个 chunk 只包含增量内容。

**解决：** 如果需要完整内容，使用 `model.invoke()` 而非 `model.stream()`，或自行拼接 chunk。
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/02-messages-and-models.md
git commit -m "docs(langchain): add chapter 02 - messages and models"
```

---

### Task 3: 第03章 工具

**Files:**
- Create: `docs/book/03-tools.md`

- [ ] **Step 1: 编写第03章**

写入 `docs/book/03-tools.md`：

```markdown
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
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/03-tools.md
git commit -m "docs(langchain): add chapter 03 - tools"
```

---

### Task 4: 第04章 智能体

**Files:**
- Create: `docs/book/04-agents.md`

- [ ] **Step 1: 编写第04章**

写入 `docs/book/04-agents.md`：

```markdown
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
| `interrupt_after` | `list[str] | None` | 在指定节点后中断 |
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
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/04-agents.md
git commit -m "docs(langchain): add chapter 04 - agents"
```

---

### Task 5: 第05章 中间件概览

**Files:**
- Create: `docs/book/05-middleware-overview.md`

- [ ] **Step 1: 编写第05章**

写入 `docs/book/05-middleware-overview.md`：

```markdown
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
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/05-middleware-overview.md
git commit -m "docs(langchain): add chapter 05 - middleware overview"
```

---

### Task 6: 第06章 内置中间件

**Files:**
- Create: `docs/book/06-built-in-middleware.md`

- [ ] **Step 1: 编写第06章**

写入 `docs/book/06-built-in-middleware.md`：

```markdown
# 第06章：内置中间件

**学习目标：** 掌握每个内置中间件的用途、配置和组合方式。

LangChain v1 提供了 14 个内置中间件，覆盖常见的智能体增强需求。

## HumanInTheLoopMiddleware — 人工审批

在执行工具前暂停，等待人工审批。

```python
from langchain.agents.middleware import HumanInTheLoopMiddleware

# 对所有工具调用要求审批
hitl = HumanInTheLoopMiddleware(
    interrupt_on={"*": True}
)

# 仅对特定工具要求审批，并允许编辑参数
hitl = HumanInTheLoopMiddleware(
    interrupt_on={
        "send_email": {"allowed_decisions": ["approve", "edit", "reject"]},
        "delete_file": {"allowed_decisions": ["approve", "reject"]},
    }
)
```

`interrupt_on` 是一个字典，键为工具名（`"*"` 表示所有工具），值为：
- `True`：仅允许批准
- `InterruptOnConfig`：配置允许的操作类型和描述

审批操作类型：`"approve"`（批准）、`"edit"`（编辑参数）、`"reject"`（拒绝）。

## ModelRetryMiddleware — 模型调用重试

模型调用失败时自动重试。

```python
from langchain.agents.middleware import ModelRetryMiddleware

retry = ModelRetryMiddleware(
    max_retries=2,            # 最多重试 2 次
    retry_on=(Exception,),    # 哪些异常触发重试
    on_failure="continue",    # 重试用尽后的行为："error" 或 "continue"
    backoff_factor=2.0,       # 退避因子
    initial_delay=1.0,        # 初始延迟（秒）
    max_delay=60.0,           # 最大延迟（秒）
    jitter=True,              # 是否添加随机抖动
)
```

`retry_on` 可以是异常类型元组或判断函数：
```python
# 仅对特定异常重试
retry = ModelRetryMiddleware(retry_on=(RateLimitError, TimeoutError))

# 用自定义函数判断
retry = ModelRetryMiddleware(retry_on=lambda e: "rate" in str(e).lower())
```

## ModelFallbackMiddleware — 模型回退

主模型失败时自动切换到备用模型。

```python
from langchain.agents.middleware import ModelFallbackMiddleware

fallback = ModelFallbackMiddleware(
    "anthropic:claude-sonnet-4-20250514",  # 第一个备用模型
    "openai:gpt-4o",                       # 第二个备用模型
)
```

模型按顺序尝试，直到某个模型成功返回。

## ModelCallLimitMiddleware — 限制模型调用次数

防止模型调用无限循环。

```python
from langchain.agents.middleware import ModelCallLimitMiddleware

limit = ModelCallLimitMiddleware(
    thread_limit=50,        # 整个线程最多 50 次模型调用
    run_limit=10,           # 单次运行最多 10 次模型调用
    exit_behavior="end",    # 超限行为："end"（结束）或 "error"（抛异常）
)
```

至少设置 `thread_limit` 或 `run_limit` 之一，否则会抛出 `ValueError`。

## ToolRetryMiddleware — 工具调用重试

工具执行失败时自动重试。参数与 `ModelRetryMiddleware` 相同，额外支持指定工具范围：

```python
from langchain.agents.middleware import ToolRetryMiddleware

retry = ToolRetryMiddleware(
    max_retries=2,
    tools=["search_web", "fetch_url"],  # 仅对指定工具重试，None 表示所有工具
    retry_on=(ConnectionError,),
    on_failure="continue",
    backoff_factor=2.0,
    initial_delay=1.0,
    max_delay=60.0,
    jitter=True,
)
```

## ToolCallLimitMiddleware — 限制工具调用次数

防止工具调用无限循环。

```python
from langchain.agents.middleware import ToolCallLimitMiddleware

limit = ToolCallLimitMiddleware(
    tool_name="search_web",  # 限制特定工具，None 表示所有限制
    thread_limit=20,         # 线程级别限制
    run_limit=5,             # 运行级别限制
    exit_behavior="continue",  # "continue"（阻止超限调用，其他继续）、"error"、"end"
)
```

## LLMToolSelectorMiddleware — 智能工具选择

当工具数量很多时，用 LLM 筛选最相关的工具子集，减少 token 消耗。

```python
from langchain.agents.middleware import LLMToolSelectorMiddleware

selector = LLMToolSelectorMiddleware(
    model="anthropic:claude-sonnet-4-20250514",  # 用于筛选的模型
    max_tools=5,                 # 最多选择 5 个工具
    always_include=["search"],  # 始终包含的工具
)
```

## LLMToolEmulator — 工具模拟器

用 LLM 模拟工具执行结果，用于测试或没有真实工具的场景。

```python
from langchain.agents.middleware import LLMToolEmulator

emulator = LLMToolEmulator(
    tools=["search_web", "calculator"],  # 要模拟的工具，None 表示所有
    model="anthropic:claude-sonnet-4-20250514",  # 用于模拟的模型，None 用主模型
)
```

## ContextEditingMiddleware — 上下文编辑

在上下文过长时自动裁剪历史消息，节省 token。

```python
from langchain.agents.middleware import ContextEditingMiddleware, ClearToolUsesEdit

# 使用默认策略（清除旧的工具调用结果）
editor = ContextEditingMiddleware(
    edits=[ClearToolUsesEdit(
        trigger=100_000,        # 超过 10 万字符时触发
        clear_at_least=0,       # 至少清除多少
        keep=3,                  # 保留最近 3 次工具调用结果
        clear_tool_inputs=False, # 是否清除工具输入
        exclude_tools=["search"],  # 排除的工具
    )],
    token_count_method="approximate",  # "approximate" 或 "model"
)
```

### ClearToolUsesEdit 参数

| 参数 | 默认值 | 说明 |
|------|--------|------|
| `trigger` | `100_000` | 触发阈值（字符数） |
| `clear_at_least` | `0` | 至少清除的量 |
| `keep` | `3` | 保留最近 N 次工具结果 |
| `clear_tool_inputs` | `False` | 是否清除工具输入参数 |
| `exclude_tools` | `()` | 排除的工具名列表 |
| `placeholder` | `"[cleared]"` | 替换被清除内容的占位符 |

## PIIMiddleware — 敏感信息检测

检测和处理对话中的个人敏感信息。

```python
from langchain.agents.middleware import PIIMiddleware

pii = PIIMiddleware(
    pii_type="email",         # 检测类型："email"、"credit_card"、"ip"、"mac_address"、"url"
    strategy="redact",        # 处理策略："block"、"redact"、"mask"、"hash"
    apply_to_input=True,      # 对用户输入应用
    apply_to_output=False,    # 对模型输出应用
    apply_to_tool_results=False,  # 对工具结果应用
)
```

### 处理策略

| 策略 | 行为 |
|------|------|
| `"block"` | 发现 PII 时阻止整个请求 |
| `"redact"` | 替换为 `[REDACTED]` |
| `"mask"` | 部分遮掩，如 `j***@gmail.com` |
| `"hash"` | 替换为哈希值 |

## ShellToolMiddleware — Shell 工具

让智能体执行 Shell 命令，支持多种安全策略。

```python
from langchain.agents.middleware import ShellToolMiddleware, HostExecutionPolicy

shell = ShellToolMiddleware(
    workspace_root="/tmp/workspace",  # 工作目录
    execution_policy=HostExecutionPolicy(),  # 执行策略
    redaction_rules=[RedactionRule(pattern=r"password=\S+", replacement="[REDACTED]")],
)
```

### 执行策略

| 策略 | 安全级别 | 说明 |
|------|---------|------|
| `HostExecutionPolicy` | 低 | 直接在主机执行（默认） |
| `DockerExecutionPolicy` | 中 | 在 Docker 容器中执行 |
| `CodexSandboxExecutionPolicy` | 高 | 在沙盒中执行 |

```python
from langchain.agents.middleware import DockerExecutionPolicy

# Docker 策略
shell = ShellToolMiddleware(
    execution_policy=DockerExecutionPolicy(image="python:3.12"),
)
```

## SummarizationMiddleware — 对话摘要

当对话过长时自动摘要历史消息。

```python
from langchain.agents.middleware import SummarizationMiddleware

summary = SummarizationMiddleware(
    model="anthropic:claude-sonnet-4-20250514",  # 用于生成摘要的模型
    trigger=("messages", 50),     # 超过 50 条消息时触发
    keep=("messages", 20),       # 保留最近 20 条消息
    trim_tokens_to_summarize=4000,  # 摘要最大 token 数
)
```

`trigger` 和 `keep` 支持三种格式：
- `("fraction", 0.8)` — 占模型上下文窗口的比例
- `("tokens", 8000)` — 绝对 token 数
- `("messages", 50)` — 绝对消息条数

## TodoListMiddleware — 任务列表

让智能体维护和跟踪任务列表。

```python
from langchain.agents.middleware import TodoListMiddleware

todo = TodoListMiddleware(
    system_prompt="你的自定义提示词",    # 覆盖默认提示
    tool_description="管理任务列表",       # 覆盖工具描述
)
```

中间件会注册 `write_todos` 工具，智能体可以创建、更新和完成任务。任务状态有三种：`"pending"`、`"in_progress"`、`"completed"`。

## FilesystemFileSearchMiddleware — 文件搜索

让智能体在工作目录中搜索文件。

```python
from langchain.agents.middleware import FilesystemFileSearchMiddleware

file_search = FilesystemFileSearchMiddleware(
    root_path="/path/to/project",  # 搜索根目录
    use_ripgrep=True,              # 使用 ripgrep 加速
    max_file_size_mb=10,           # 跳过超过 10MB 的文件
)
```

注册 `glob_search` 和 `grep_search` 两个工具。

## 中间件组合

多个中间件可以自由组合，按列表顺序包裹：

```python
from langchain.agents import create_agent

agent = create_agent(
    model,
    tools=[search_tool],
    middleware=[
        ModelCallLimitMiddleware(run_limit=10),  # 最外层：限制调用
        ModelRetryMiddleware(max_retries=2),      # 重试
        PIIMiddleware("email", strategy="redact"), # 检测 PII
    ],
)
```

### 组合建议

- **生产环境必备：** `ModelCallLimitMiddleware` + `ModelRetryMiddleware`
- **处理敏感数据：** 加上 `PIIMiddleware`
- **长对话：** 加上 `SummarizationMiddleware` 或 `ContextEditingMiddleware`
- **需要人工确认：** 加上 `HumanInTheLoopMiddleware`

## 下一步

- [第07章：结构化输出](07-structured-output.md) — 让智能体返回结构化数据
- [第05章：中间件概览](05-middleware-overview.md) — 编写自定义中间件

## 常见问题

### 中间件顺序重要吗？

**是的。** 外层中间件（列表前面）先处理 `before` 钩子，后处理 `after` 钩子。`wrap_model_call` 按洋葱模型嵌套。把全局限制放在前面，具体处理放在后面。

### 多个 wrap_model_call 中间件如何交互？

它们按洋葱模型嵌套。列表中第一个 `wrap_model_call` 最外层，最后一个最内层（最靠近模型）。`handler(request)` 会调用下一层中间件。

### PIIMiddleware 的 detector 参数怎么用？

`detector` 支持三种值：
- `None`（默认）：使用内置正则检测器
- 字符串：指定检测器名称
- 函数：自定义检测函数，签名为 `(text: str) -> list[PIIMatch]`
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/06-built-in-middleware.md
git commit -m "docs(langchain): add chapter 06 - built-in middleware"
```

---

### Task 7: 第07章 结构化输出

**Files:**
- Create: `docs/book/07-structured-output.md`

- [ ] **Step 1: 编写第07章**

写入 `docs/book/07-structured-output.md`：

```markdown
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
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/07-structured-output.md
git commit -m "docs(langchain): add chapter 07 - structured output"
```

---

### Task 8: 第08章 嵌入模型

**Files:**
- Create: `docs/book/08-embeddings.md`

- [ ] **Step 1: 编写第08章**

写入 `docs/book/08-embeddings.md`：

```markdown
# 第08章：嵌入模型

**学习目标：** 使用 `init_embeddings()` 获取文本向量表示。

## 什么是嵌入？

嵌入（Embeddings）将文本转换为高维向量——一组数字。语义相似的文本会产生相近的向量。嵌入常用于语义搜索、文本分类和聚类。

## init_embeddings()

`init_embeddings()` 是初始化嵌入模型的统一入口。

### 完整签名

```python
def init_embeddings(
    model: str,
    *,
    provider: str | None = None,
    **kwargs: Any,
) -> Embeddings
```

### 基本用法

```python
from langchain.embeddings import init_embeddings

# 指定模型和提供商
embeddings = init_embeddings("text-embedding-3-small", provider="openai")

# 使用 "提供商:模型" 格式
embeddings = init_embeddings("openai:text-embedding-3-small")
```

### 内置提供商

| 提供商 | 包名 |
|--------|------|
| `openai` | `langchain-openai` |
| `azure_openai` | `langchain-openai` |
| `azure_ai` | `langchain-azure-ai` |
| `bedrock` | `langchain-aws` |
| `cohere` | `langchain-cohere` |
| `google_genai` | `langchain-google-genai` |
| `google_vertexai` | `langchain-google-vertexai` |
| `huggingface` | `langchain-huggingface` |
| `mistralai` | `langchain-mistralai` |
| `ollama` | `langchain-ollama` |

## 嵌入与聊天模型的区别

| 特性 | 嵌入模型 | 聊天模型 |
|------|---------|---------|
| 输入 | 文本 | 消息列表 |
| 输出 | 浮点数向量 | 文本/工具调用 |
| 用途 | 搜索、分类、聚类 | 对话、推理、工具使用 |
| 初始化 | `init_embeddings()` | `init_chat_model()` |

## 生成嵌入

```python
embeddings = init_embeddings("openai:text-embedding-3-small")

# 嵌入单个文本
vector = embeddings.embed_query("什么是机器学习？")
print(f"向量维度：{len(vector)}")  # 例如 1536

# 嵌入多个文档
docs = ["Python 是一种编程语言", "Rust 是一种系统编程语言"]
vectors = embeddings.embed_documents(docs)
print(f"生成了 {len(vectors)} 个向量")
```

## 异步嵌入

```python
# 异步嵌入单个文本
vector = await embeddings.aembed_query("什么是机器学习？")

# 异步嵌入多个文档
vectors = await embeddings.aembed_documents(docs)
```

## 与向量数据库配合

嵌入最常见的用途是与向量数据库配合实现语义搜索：

```python
from langchain.embeddings import init_embeddings

embeddings = init_embeddings("openai:text-embedding-3-small")

# 伪代码——实际使用需要安装对应的向量数据库
# 1. 将文档存入向量数据库
# vectorstore.add_texts(["文档1内容", "文档2内容"], embeddings=embeddings)

# 2. 语义搜索
# results = vectorstore.similarity_search("查询内容", embeddings=embeddings)
```

具体的向量数据库使用方法，参见各 provider 的文档。

## 下一步

- [第09章：速率限制](09-rate-limiters.md) — 保护 API 调用
- [第02章：消息与模型](02-messages-and-models.md) — 回顾聊天模型

## 常见问题

### embed_query 和 embed_documents 的区别

**原因：** 两者用途不同。

**解决：** `embed_query` 用于嵌入用户查询（单个文本），`embed_documents` 用于嵌入文档列表（批量）。某些提供商对两者有不同的优化。

### 嵌入维度不一致

**原因：** 不同的嵌入模型产生不同维度的向量。

**解决：** 确保存储和查询使用同一个模型。混合使用不同模型的向量会导致搜索结果不准确。
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/08-embeddings.md
git commit -m "docs(langchain): add chapter 08 - embeddings"
```

---

### Task 9: 第09章 速率限制

**Files:**
- Create: `docs/book/09-rate-limiters.md`

- [ ] **Step 1: 编写第09章**

写入 `docs/book/09-rate-limiters.md`：

```markdown
# 第09章：速率限制

**学习目标：** 使用内置速率限制器保护 API 调用。

## 为什么需要速率限制？

大多数 API 提供商都有请求频率限制。超过限制会导致请求失败或被临时封禁。速率限制器帮你控制请求频率，避免触发提供商的限制。

## InMemoryRateLimiter

LangChain 提供基于内存的速率限制器：

```python
from langchain.rate_limiters import InMemoryRateLimiter

# 创建速率限制器
rate_limiter = InMemoryRateLimiter(
    requests_per_second=5,   # 每秒最多 5 个请求
    check_every_n_seconds=0.1,  # 检查间隔
    max_bucket_size=10,      # 令牌桶最大容量
)
```

### 参数说明

| 参数 | 类型 | 说明 |
|------|------|------|
| `requests_per_second` | `float` | 每秒允许的最大请求数 |
| `check_every_n_seconds` | `float` | 令牌桶检查间隔 |
| `max_bucket_size` | `int` | 令牌桶最大容量，允许短时突发 |

### 令牌桶算法

`InMemoryRateLimiter` 使用令牌桶算法：
- 令牌以 `requests_per_second` 的速率填充桶
- 每个请求消耗一个令牌
- 桶空时，新请求等待直到有令牌可用
- `max_bucket_size` 允许短时突发请求

## 与 init_chat_model() 配合

将速率限制器传给模型：

```python
from langchain.chat_models import init_chat_model
from langchain.rate_limiters import InMemoryRateLimiter

rate_limiter = InMemoryRateLimiter(requests_per_second=5)

model = init_chat_model(
    "claude-sonnet-4-20250514",
    model_provider="anthropic",
    rate_limiter=rate_limiter,
)

# 所有通过此模型的请求都受速率限制
result = model.invoke("你好")
```

## 并发控制

速率限制器同时控制并发请求数：

```python
# 严格限制：每秒 1 个请求
strict_limiter = InMemoryRateLimiter(
    requests_per_second=1,
    max_bucket_size=1,  # 不允许突发
)

# 宽松限制：每秒 10 个请求，允许短时突发到 20
loose_limiter = InMemoryRateLimiter(
    requests_per_second=10,
    max_bucket_size=20,
)
```

## 自定义速率限制器

继承 `BaseRateLimiter` 实现自定义逻辑：

```python
from langchain.rate_limiters import BaseRateLimiter

class CustomRateLimiter(BaseRateLimiter):
    """自定义速率限制器示例。"""

    def acquire(self) -> None:
        """同步获取一个请求许可。阻塞直到可用。"""
        # 实现自定义获取逻辑
        ...

    async def aacquire(self) -> None:
        """异步获取一个请求许可。阻塞直到可用。"""
        # 实现自定义异步获取逻辑
        ...
```

## 下一步

- [第10章：API 参考](10-api-reference.md) — 完整 API 索引
- [第02章：消息与模型](02-messages-and-models.md) — 回顾聊天模型

## 常见问题

### 速率限制器导致请求变慢

**原因：** `requests_per_second` 设置过低，或 `max_bucket_size` 太小。

**解决：** 根据提供商的实际限制调整参数。大部分提供商允许每秒 5-10 个请求。

### 多个模型共享速率限制器

**原因：** 同一提供商的多个模型共享 API 配额。

**解决：** 创建一个速率限制器实例，传给所有使用同一提供商的模型：

```python
limiter = InMemoryRateLimiter(requests_per_second=5)
model1 = init_chat_model("gpt-4o", rate_limiter=limiter)
model2 = init_chat_model("gpt-4o-mini", rate_limiter=limiter)
```
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/09-rate-limiters.md
git commit -m "docs(langchain): add chapter 09 - rate limiters"
```

---

### Task 10: 第10章 API 参考

**Files:**
- Create: `docs/book/10-api-reference.md`

- [ ] **Step 1: 编写第10章**

写入 `docs/book/10-api-reference.md`：

```markdown
# 第10章：API 参考

**学习目标：** 快速查找任何公共 API 的签名和参数说明。

本章是 `langchain_v1` 全部公共 API 的索引。详细说明请参考各章节或[在线 API 参考](https://reference.langchain.com/python/langchain/langchain/)。

## langchain.agents

### create_agent

```python
def create_agent(
    model: str | BaseChatModel,
    tools: Sequence[BaseTool | Callable | dict] | None = None,
    *,
    system_prompt: str | SystemMessage | None = None,
    middleware: Sequence[AgentMiddleware] = (),
    response_format: ResponseFormat | type | dict | None = None,
    state_schema: type[AgentState] | None = None,
    context_schema: type | None = None,
    checkpointer: Checkpointer | None = None,
    store: BaseStore | None = None,
    interrupt_before: list[str] | None = None,
    interrupt_after: list[str] | None = None,
    debug: bool = False,
    name: str | None = None,
    cache: BaseCache | None = None,
) -> CompiledStateGraph
```

详见[第04章：智能体](04-agents.md)。

### AgentState

```python
class AgentState(TypedDict, Generic[ResponseT]):
    messages: Required[Annotated[list[AnyMessage], add_messages]]
    jump_to: NotRequired[Annotated[JumpTo | None, EphemeralValue, PrivateStateAttr]]
    structured_response: NotRequired[Annotated[ResponseT, OmitFromInput]]
```

详见[第04章：智能体](04-agents.md)。

## langchain.agents.middleware

### AgentMiddleware

```python
class AgentMiddleware(Generic[StateT, ContextT, ResponseT]):
    state_schema: type[StateT]
    tools: Sequence[BaseTool]
    name: str  # 属性
```

详见[第05章：中间件概览](05-middleware-overview.md)。

### 装饰器

| 装饰器 | 签名 | 详见 |
|--------|------|------|
| `before_model` | `(func, *, state_schema, tools, can_jump_to, name)` | [第05章](05-middleware-overview.md) |
| `after_model` | `(func, *, state_schema, tools, can_jump_to, name)` | [第05章](05-middleware-overview.md) |
| `before_agent` | `(func, *, state_schema, tools, can_jump_to, name)` | [第05章](05-middleware-overview.md) |
| `after_agent` | `(func, *, state_schema, tools, can_jump_to, name)` | [第05章](05-middleware-overview.md) |
| `wrap_model_call` | `(func, *, state_schema, tools, name)` | [第05章](05-middleware-overview.md) |
| `wrap_tool_call` | `(func, *, tools, name)` | [第05章](05-middleware-overview.md) |
| `dynamic_prompt` | `(func)` | [第05章](05-middleware-overview.md) |
| `hook_config` | `(*, can_jump_to)` | [第05章](05-middleware-overview.md) |

### 类型

| 类型 | 说明 |
|------|------|
| `ModelRequest` | 模型调用请求，含 `model`、`messages`、`system_message`、`tools`、`state`、`runtime` 等 |
| `ModelResponse` | 模型调用响应，含 `result`、`structured_response` |
| `ExtendedModelResponse` | 扩展响应，含 `model_response`、`command` |
| `ModelCallResult` | `ModelResponse | AIMessage | ExtendedModelResponse` |
| `ToolCallRequest` | 工具调用请求 |
| `JumpTo` | `Literal["tools", "model", "end"]` |
| `ResponseT` | 结构化输出类型变量，默认 `Any` |
| `ContextT` | 运行时上下文类型变量 |
| `OmitFromSchema` | 控制字段在 schema 中的可见性 |

### 内置中间件

| 中间件 | 签名 | 详见 |
|--------|------|------|
| `HumanInTheLoopMiddleware` | `(interrupt_on, *, description_prefix)` | [第06章](06-built-in-middleware.md) |
| `ModelRetryMiddleware` | `(*, max_retries, retry_on, on_failure, backoff_factor, initial_delay, max_delay, jitter)` | [第06章](06-built-in-middleware.md) |
| `ModelFallbackMiddleware` | `(first_model, *additional_models)` | [第06章](06-built-in-middleware.md) |
| `ModelCallLimitMiddleware` | `(*, thread_limit, run_limit, exit_behavior)` | [第06章](06-built-in-middleware.md) |
| `ToolRetryMiddleware` | `(*, max_retries, tools, retry_on, on_failure, backoff_factor, initial_delay, max_delay, jitter)` | [第06章](06-built-in-middleware.md) |
| `ToolCallLimitMiddleware` | `(*, tool_name, thread_limit, run_limit, exit_behavior)` | [第06章](06-built-in-middleware.md) |
| `LLMToolSelectorMiddleware` | `(*, model, system_prompt, max_tools, always_include)` | [第06章](06-built-in-middleware.md) |
| `LLMToolEmulator` | `(*, tools, model)` | [第06章](06-built-in-middleware.md) |
| `ContextEditingMiddleware` | `(*, edits, token_count_method)` | [第06章](06-built-in-middleware.md) |
| `PIIMiddleware` | `(pii_type, *, strategy, detector, apply_to_input, apply_to_output, apply_to_tool_results)` | [第06章](06-built-in-middleware.md) |
| `ShellToolMiddleware` | `(workspace_root, *, startup_commands, shutdown_commands, execution_policy, redaction_rules, ...)` | [第06章](06-built-in-middleware.md) |
| `SummarizationMiddleware` | `(model, *, trigger, keep, token_counter, summary_prompt, trim_tokens_to_summarize)` | [第06章](06-built-in-middleware.md) |
| `TodoListMiddleware` | `(*, system_prompt, tool_description)` | [第06章](06-built-in-middleware.md) |
| `FilesystemFileSearchMiddleware` | `(*, root_path, use_ripgrep, max_file_size_mb)` | [第06章](06-built-in-middleware.md) |

## langchain.agents.structured_output

### ResponseFormat

```python
ResponseFormat = ToolStrategy[SchemaT] | ProviderStrategy[SchemaT] | AutoStrategy[SchemaT]
```

详见[第07章：结构化输出](07-structured-output.md)。

### 策略类

| 类 | 签名 | 详见 |
|----|------|------|
| `AutoStrategy` | `(schema)` | [第07章](07-structured-output.md) |
| `ProviderStrategy` | `(schema, *, strict)` | [第07章](07-structured-output.md) |
| `ToolStrategy` | `(schema, *, tool_message_content, handle_errors)` | [第07章](07-structured-output.md) |

### 错误类

| 类 | 说明 |
|----|------|
| `StructuredOutputError` | 基类，含 `ai_message: AIMessage` |
| `MultipleStructuredOutputsError` | 多个结构化输出，含 `tool_names: list[str]` |
| `StructuredOutputValidationError` | 验证失败，含 `tool_name: str`、`source: Exception` |

## langchain.chat_models

### init_chat_model

```python
def init_chat_model(
    model: str | None = None,
    *,
    model_provider: str | None = None,
    configurable_fields: Literal["any"] | list[str] | tuple[str, ...] | None = None,
    config_prefix: str | None = None,
    **kwargs: Any,
) -> BaseChatModel | _ConfigurableModel
```

详见[第02章：消息与模型](02-messages-and-models.md)。

## langchain.embeddings

### init_embeddings

```python
def init_embeddings(
    model: str,
    *,
    provider: str | None = None,
    **kwargs: Any,
) -> Embeddings
```

详见[第08章：嵌入模型](08-embeddings.md)。

## langchain.messages

| 导出名 | 类型 | 说明 |
|--------|------|------|
| `HumanMessage` | 类 | 用户消息 |
| `AIMessage` | 类 | AI 回复 |
| `AIMessageChunk` | 类 | AI 回复片段（流式） |
| `SystemMessage` | 类 | 系统指令 |
| `ToolMessage` | 类 | 工具执行结果 |
| `RemoveMessage` | 类 | 删除指定消息 |
| `AnyMessage` | 类型别名 | 任意消息类型 |
| `TextContentBlock` | 类 | 文本内容块 |
| `ImageContentBlock` | 类 | 图片内容块 |
| `AudioContentBlock` | 类 | 音频内容块 |
| `VideoContentBlock` | 类 | 视频内容块 |
| `FileContentBlock` | 类 | 文件内容块 |
| `DataContentBlock` | 类 | 数据内容块 |
| `ReasoningContentBlock` | 类 | 推理过程内容块 |
| `ToolCall` | 类 | 工具调用描述 |
| `ToolCallChunk` | 类 | 工具调用片段 |
| `UsageMetadata` | 类 | Token 使用信息 |
| `trim_messages` | 函数 | 裁剪消息列表 |

详见[第02章：消息与模型](02-messages-and-models.md)。

## langchain.rate_limiters

| 导出名 | 类型 | 说明 |
|--------|------|------|
| `BaseRateLimiter` | 抽象类 | 速率限制器基类 |
| `InMemoryRateLimiter` | 类 | 基于内存的速率限制器 |

详见[第09章：速率限制](09-rate-limiters.md)。

## langchain.tools

| 导出名 | 类型 | 说明 |
|--------|------|------|
| `ToolNode` | 类 | 工具执行节点 |
| `InjectedState` | 类 | 状态注入标注 |
| `InjectedStore` | 类 | 存储注入标注 |
| `ToolRuntime` | 类 | 运行时注入标注 |
| `ToolCallRequest` | 类 | 工具调用请求 |
| `ToolCallWithContext` | 类 | 带上下文的工具调用 |
| `ToolCallWrapper` | 类 | 工具调用包装器 |

详见[第03章：工具](03-tools.md)。
```

- [ ] **Step 2: 提交**

```bash
git add libs/langchain_v1/docs/book/10-api-reference.md
git commit -m "docs(langchain): add chapter 10 - api reference"
```

---

### Task 11: 最终验证和汇总提交

**Files:**
- Verify: `docs/book/` 目录下所有 10 个文件

- [ ] **Step 1: 验证所有文件存在**

```bash
ls -la libs/langchain_v1/docs/book/
```

预期输出：10 个编号的 Markdown 文件。

- [ ] **Step 2: 验证文件间链接正确**

检查每个章节中的相对链接（如 `[第02章](02-messages-and-models.md)`）指向存在的文件。

```bash
cd libs/langchain_v1/docs/book && grep -oh '\[.*\](.*\.md)' *.md | sort
```

- [ ] **Step 3: 验证术语一致性**

确认所有章节使用统一术语：智能体（非代理）、中间件（非中间层）、提供商（非供应商）等。

```bash
cd libs/langchain_v1/docs/book && grep -c '代理\|中间层\|供应商' *.md
```

预期输出：所有计数为 0。

- [ ] **Step 4: 汇总提交**

如果有任何修正，提交最终版本：

```bash
git add libs/langchain_v1/docs/book/
git commit -m "docs(langchain): complete v1 book - 10 chapters covering full API"
```