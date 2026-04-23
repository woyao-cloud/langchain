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