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