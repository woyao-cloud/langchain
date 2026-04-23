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