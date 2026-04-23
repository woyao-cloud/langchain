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