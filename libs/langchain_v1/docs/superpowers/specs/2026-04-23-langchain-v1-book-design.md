---
name: LangChain v1 Book Design
description: 中文文档书设计，面向初学者，覆盖 langchain_v1 全包 API
type: project
---

# LangChain v1 文档书设计

## 概述

为 `langchain_v1` 包编写一本面向初学者的中文文档书，覆盖全部公共 API。采用概念优先（Concept-first）的结构组织，从简单到复杂递进。

## 目标读者

Python 开发者，初次接触 LangChain v1，希望从零学习 v1 API。

## 书写语言

全部中文，包括代码注释。代码变量名保持英文。

## 格式

仓库内 Markdown 文件，存放在 `libs/langchain_v1/docs/book/` 目录。

## 内容深度

四个层次全部覆盖：
- 概念说明
- 代码示例
- 教程/演练
- API 参考

## 章节结构

| # | 文件 | 范围 |
|---|------|------|
| 01 | `getting-started.md` | 安装、第一个智能体、5分钟快速开始 |
| 02 | `messages-and-models.md` | 消息类型体系、`init_chat_model()`、provider 注册表、流式输出 |
| 03 | `tools.md` | `@tool`、`StructuredTool`、`ToolNode`、`InjectedState`、`ToolRuntime` |
| 04 | `agents.md` | `create_agent()`、`AgentState`、执行循环、`Runtime`、控制流 |
| 05 | `middleware-overview.md` | 中间件理念、`AgentMiddleware`、六个装饰器钩子、自定义中间件 |
| 06 | `built-in-middleware.md` | 每个内置中间件的用法和配置 |
| 07 | `structured-output.md` | `ResponseFormat`、三种策略、错误处理 |
| 08 | `embeddings.md` | `init_embeddings()`、provider 注册表、批量嵌入 |
| 09 | `rate-limiters.md` | `InMemoryRateLimiter`、自定义速率限制器 |
| 10 | `api-reference.md` | 按模块分组的公共 API 索引 |

## 各章节内容设计

### 第01章：快速开始 (`01-getting-started.md`)

**学习目标：** 5分钟内创建并运行第一个 LangChain v1 智能体。

- 安装：`pip install langchain` 和可选 provider 包
- 配置 API Key
- 最小可运行示例：创建带工具的智能体并对话
- `create_agent()` 的最简调用方式
- 智能体输出结果的基本解读
- 下一步指引：第02章

**代码示例风格：**
```python
# 安装核心包和 Anthropic 提供商
# pip install langchain[anthropic]

import os
from langchain.agents import create_agent
from langchain.chat_models import init_chat_model

# 设置 API 密钥
os.environ["ANTHROPIC_API_KEY"] = "your-key-here"

# 初始化聊天模型
model = init_chat_model("claude-sonnet-4-20250514", model_provider="anthropic")

# 创建并运行智能体
agent = create_agent(model)
result = agent.invoke({"messages": [("user", "你好，请介绍一下你自己")]})
print(result)
```

### 第02章：消息与模型 (`02-messages-and-models.md`)

**学习目标：** 理解消息系统，掌握 `init_chat_model()` 全部用法。

- 消息类型体系：`HumanMessage`、`AIMessage`、`SystemMessage`、`ToolMessage`、`RemoveMessage`
- 内容块：`TextContentBlock`、`ImageContentBlock`、`AudioContentBlock` 等
- `init_chat_model()` 详解：自动推断 provider、显式指定、配置参数
- 内置 provider 注册表
- 流式输出基础
- `UsageMetadata` 和 token 计量

### 第03章：工具 (`03-tools.md`)

**学习目标：** 定义和注册工具，让智能体与外部世界交互。

- `@tool` 装饰器定义工具
- `StructuredTool` 类定义工具
- `ToolNode` 和 `InjectedState`、`InjectedStore`、`ToolRuntime`
- 工具的错误处理
- `ToolCallRequest` 和 `ToolCallWithContext`
- 动态工具选择（与 `LLMToolSelectorMiddleware` 的衔接）

### 第04章：智能体 (`04-agents.md`)

**学习目标：** 深入理解 `create_agent()` 和 `AgentState`，掌握智能体完整生命周期。

- `create_agent()` 完整签名和所有参数
- `AgentState` 类型定义：自定义状态字段、`OmitFromSchema`
- 智能体执行循环（模型调用 → 工具调用 → 模型调用…）
- `Runtime` 和状态持久化
- `Command`、`Send` 等控制流原语
- `JumpTo` 跳转控制
- 编译图（`CompiledStateGraph`）与 LangGraph 的关系

### 第05章：中间件概览 (`05-middleware-overview.md`)

**学习目标：** 理解中间件核心理念和装饰器 API，能够编写自定义中间件。

- 中间件概念：拦截和修改智能体行为的插件机制
- `AgentMiddleware` 协议
- 六个装饰器钩子详解：
  - `before_model` / `after_model`：模型调用前后
  - `before_agent` / `after_agent`：智能体步骤前后
  - `wrap_model_call`：包装模型调用
  - `wrap_tool_call`：包装工具调用
- `ModelRequest`、`ModelResponse`、`ExtendedModelResponse`
- `dynamic_prompt` 动态修改系统提示
- `hook_config` 配置钩子
- 编写第一个自定义中间件

### 第06章：内置中间件 (`06-built-in-middleware.md`)

**学习目标：** 掌握每个内置中间件的用途、配置和组合方式。

每个中间件一个子节：
- `HumanInTheLoopMiddleware`：人工审批与中断配置
- `ModelRetryMiddleware`：模型调用自动重试
- `ModelFallbackMiddleware`：模型回退策略
- `ModelCallLimitMiddleware`：限制模型调用次数
- `ToolRetryMiddleware`：工具调用重试
- `ToolCallLimitMiddleware`：限制工具调用次数
- `LLMToolSelectorMiddleware`：智能工具选择
- `LLMToolEmulator`：工具模拟器
- `ContextEditingMiddleware` / `ClearToolUsesEdit`：上下文编辑
- `PIIMiddleware`：敏感信息检测与脱敏
- `ShellToolMiddleware`：Shell 工具安全策略
- `SummarizationMiddleware`：长对话自动摘要
- `TodoListMiddleware`：任务列表管理
- `FilesystemFileSearchMiddleware`：文件搜索
- 中间件组合最佳实践

### 第07章：结构化输出 (`07-structured-output.md`)

**学习目标：** 让智能体返回结构化数据而非自由文本。

- `ResponseFormat` 定义（Pydantic、dataclass、TypedDict、JSON Schema）
- 三种策略：`AutoStrategy`、`ProviderStrategy`、`ToolStrategy`
- `ProviderStrategyBinding` 和 `OutputToolBinding`
- 错误处理：`StructuredOutputError`、`StructuredOutputValidationError`、`MultipleStructuredOutputsError`
- 与 `create_agent()` 集成

### 第08章：嵌入模型 (`08-embeddings.md`)

**学习目标：** 使用 `init_embeddings()` 获取文本向量表示。

- `init_embeddings()` 用法：自动推断 provider、显式指定
- 内置 provider 注册表
- 嵌入模型与聊天模型的区别
- 批量嵌入和异步嵌入
- 与向量数据库配合的基础模式

### 第09章：速率限制 (`09-rate-limiters.md`)

**学习目标：** 使用内置速率限制器保护 API 调用。

- `InMemoryRateLimiter` 的创建和配置
- 与 `init_chat_model()` 配合使用
- 请求速率和并发控制
- 自定义速率限制器（继承 `BaseRateLimiter`）

### 第10章：API 参考 (`10-api-reference.md`)

**学习目标：** 快速查找任何公共 API 的签名和参数说明。

- 按模块分组的 API 索引
- 每个公共函数/类的签名、参数说明和返回值
- 链接到官方在线参考文档
- 关键类型别名说明（`JumpTo`、`ResponseT`、`ContextT` 等）

## 跨章节设计决策

### 代码示例规范

- **Python 版本：** 3.10+
- **导入风格：** 统一使用 `from langchain.xxx import Yyy`，不使用 `from langchain_core.xxx`
- **API Key 管理：** 示例使用 `os.environ`，附注说明生产环境应使用 `.env` 或密钥管理服务
- **异步示例：** 先给出同步版本，再给出 `async` 版本，用独立代码块标注
- **输出展示：** 用 `# 输出：` 注释展示预期结果

### 术语约定

| 英文 | 中文翻译 | 备注 |
|------|---------|------|
| Agent | 智能体 | 全书统一 |
| Middleware | 中间件 | 全书统一 |
| Tool | 工具 | 全书统一 |
| Chat Model | 聊天模型 | 全书统一 |
| Embeddings | 嵌入（模型） | 不用"向量化" |
| Rate Limiter | 速率限制器 | 全书统一 |
| Structured Output | 结构化输出 | 全书统一 |
| Provider | 提供商 | 全书统一 |
| State | 状态 | 全书统一 |
| Hook | 钩子 | 全书统一 |
| Decorator | 装饰器 | 全书统一 |

### 章节依赖关系

```
01 → 02 → 03 → 04 → 05 → 06
                ↑        ↑
                └── 07 ──┘
08 ← 02
09 ← 02
10 ← 所有章节（被引用的索引页）
```

- 第01-04章严格递进
- 第05-06章依赖第04章
- 第07章依赖第03章和第04章
- 第08、09章可独立阅读，建议先读第02章
- 第10章是独立索引页

### 错误处理与调试

每个章节包含"常见问题"小节：
- 最常见的 2-3 个错误及原因和解决方法
- 调试技巧（启用 LangSmith 追踪、查看中间件执行流程）

### 与现有文档的关系

- 本书聚焦 `langchain_v1` 特有的高层 API，不替代 `langchain_core` 的 API 参考
- 涉及 `langchain_core` 底层概念时（如 `BaseMessage`），用简短说明 + 链接指向官方文档
- 不重复 LangGraph 教程内容，仅在必要时说明与 LangGraph 的接口关系

## 不包含的内容

- `langchain-classic`（旧版 langchain 包）的迁移指南
- LangGraph 的底层 API 教程（仅说明接口关系）
- Partner 包的内部实现细节
- 部署和运维指南