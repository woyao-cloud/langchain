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