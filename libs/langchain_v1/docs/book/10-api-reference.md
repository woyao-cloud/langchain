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