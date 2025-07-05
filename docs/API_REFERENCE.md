# AutoFlow API 参考文档

## 📋 目录

- [API概述](#api概述)
- [认证方式](#认证方式)
- [聊天API](#聊天api)
- [文档管理API](#文档管理api)
- [知识库管理API](#知识库管理api)
- [检索API](#检索api)
- [错误处理](#错误处理)

## 🌐 API概述

AutoFlow提供RESTful API，支持所有核心功能的程序化访问。API基于FastAPI构建，提供自动生成的OpenAPI文档。

### 基础信息
- **基础URL**: `https://your-domain.com/api`
- **API版本**: v1
- **数据格式**: JSON
- **字符编码**: UTF-8

### API文档地址
- **Swagger UI**: `https://your-domain.com/api-docs`
- **ReDoc**: `https://your-domain.com/redoc`

## 🔐 认证方式

AutoFlow支持多种认证方式：

### 1. API Key认证（推荐）
```bash
curl -H "Authorization: Bearer your-api-key" \
     https://your-domain.com/api/chats
```

### 2. Session认证
适用于Web应用的会话认证。

### 3. 无认证访问
某些公开API端点支持无认证访问。

## 💬 聊天API

### 发起对话

创建新的对话或继续现有对话。

**端点**: `POST /api/chats`

**请求体**:
```json
{
  "messages": [
    {
      "role": "user",
      "content": "什么是人工智能？"
    }
  ],
  "chat_engine": "default",
  "stream": true,
  "chat_id": "optional-chat-id"
}
```

**参数说明**:
- `messages` (array): 对话消息列表
  - `role` (string): 消息角色，`user` 或 `assistant`
  - `content` (string): 消息内容
- `chat_engine` (string): 使用的聊天引擎名称
- `stream` (boolean): 是否使用流式响应，默认 `true`
- `chat_id` (string, 可选): 现有对话ID，用于继续对话

**响应示例**:

流式响应（`stream: true`）:
```
data: {"event_type": "text_part", "payload": "人工智能"}
data: {"event_type": "text_part", "payload": "（AI）是"}
data: {"event_type": "message_annotations_part", "payload": {"state": "source_nodes", "context": [...]}}
data: {"event_type": "done"}
```

非流式响应（`stream: false`）:
```json
{
  "message": {
    "role": "assistant",
    "content": "人工智能（AI）是计算机科学的一个分支..."
  },
  "sources": [
    {
      "document_id": "doc_123",
      "chunk_id": "chunk_456",
      "content": "相关文档内容...",
      "score": 0.85
    }
  ],
  "chat_id": "chat_789"
}
```

### 获取对话列表

**端点**: `GET /api/chats`

**查询参数**:
- `page` (int): 页码，默认1
- `size` (int): 每页数量，默认20
- `visibility` (string): 可见性过滤，`public`、`private`或`all`

**响应示例**:
```json
{
  "items": [
    {
      "id": "chat_123",
      "title": "关于AI的讨论",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T01:00:00Z",
      "message_count": 5,
      "visibility": "private"
    }
  ],
  "total": 100,
  "page": 1,
  "size": 20,
  "pages": 5
}
```

### 获取对话详情

**端点**: `GET /api/chats/{chat_id}`

**响应示例**:
```json
{
  "chat": {
    "id": "chat_123",
    "title": "关于AI的讨论",
    "created_at": "2024-01-01T00:00:00Z",
    "updated_at": "2024-01-01T01:00:00Z",
    "visibility": "private"
  },
  "messages": [
    {
      "id": "msg_1",
      "role": "user",
      "content": "什么是人工智能？",
      "created_at": "2024-01-01T00:00:00Z"
    },
    {
      "id": "msg_2",
      "role": "assistant",
      "content": "人工智能是...",
      "created_at": "2024-01-01T00:01:00Z",
      "sources": [...]
    }
  ]
}
```

### 更新对话

**端点**: `PUT /api/chats/{chat_id}`

**请求体**:
```json
{
  "title": "新的对话标题",
  "visibility": "public"
}
```

### 删除对话

**端点**: `DELETE /api/chats/{chat_id}`

**响应**: `204 No Content`

## 📄 文档管理API

### 下载文档

**端点**: `GET /api/documents/{doc_id}/download`

**响应**: 文档文件的二进制流

## 🧠 知识库管理API

### 创建知识库

**端点**: `POST /admin/knowledge_bases`

**权限**: 需要管理员权限

**请求体**:
```json
{
  "name": "我的知识库",
  "description": "知识库描述",
  "llm_id": 1,
  "embedding_model_id": 1,
  "data_sources": [
    {
      "name": "文档源1",
      "data_source_type": "file",
      "config": {
        "file_path": "/path/to/documents"
      }
    }
  ]
}
```

**响应示例**:
```json
{
  "id": 1,
  "name": "我的知识库",
  "description": "知识库描述",
  "created_at": "2024-01-01T00:00:00Z",
  "updated_at": "2024-01-01T00:00:00Z",
  "llm": {
    "id": 1,
    "name": "gpt-4o-mini",
    "provider": "openai"
  },
  "embedding_model": {
    "id": 1,
    "name": "text-embedding-3-small",
    "provider": "openai"
  },
  "data_sources": [...],
  "stats": {
    "document_count": 0,
    "chunk_count": 0,
    "entity_count": 0,
    "relationship_count": 0
  }
}
```

### 获取知识库列表

**端点**: `GET /admin/knowledge_bases`

**权限**: 需要管理员权限

**查询参数**:
- `page` (int): 页码
- `size` (int): 每页数量

### 获取知识库详情

**端点**: `GET /admin/knowledge_bases/{kb_id}`

### 更新知识库

**端点**: `PUT /admin/knowledge_bases/{kb_id}`

### 删除知识库

**端点**: `DELETE /admin/knowledge_bases/{kb_id}`

### 获取知识库文档列表

**端点**: `GET /admin/knowledge_bases/{kb_id}/documents`

**查询参数**:
- `page` (int): 页码
- `size` (int): 每页数量
- `status` (string): 文档状态过滤

**响应示例**:
```json
{
  "items": [
    {
      "id": 1,
      "name": "document.pdf",
      "source_uri": "file://document.pdf",
      "mime_type": "application/pdf",
      "size": 1024000,
      "status": "completed",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T01:00:00Z",
      "chunk_count": 50,
      "index_status": "completed"
    }
  ],
  "total": 10,
  "page": 1,
  "size": 20
}
```

## 🔍 检索API

### 文档检索

**端点**: `POST /api/retrieve/documents`

**请求体**:
```json
{
  "query": "人工智能的应用",
  "top_k": 5,
  "knowledge_base_ids": [1, 2],
  "similarity_threshold": 0.7
}
```

**响应示例**:
```json
{
  "chunks": [
    {
      "id": "chunk_123",
      "text": "人工智能在医疗领域的应用...",
      "score": 0.92,
      "document": {
        "id": 1,
        "name": "AI应用报告.pdf",
        "source_uri": "file://ai-report.pdf"
      },
      "metadata": {
        "page": 5,
        "section": "医疗应用"
      }
    }
  ],
  "total": 50,
  "query_time": 0.15
}
```

### 知识图谱检索

**端点**: `POST /api/retrieve/knowledge_graph`

**请求体**:
```json
{
  "query": "人工智能与机器学习的关系",
  "depth": 2,
  "knowledge_base_ids": [1],
  "max_entities": 10,
  "max_relationships": 20
}
```

**响应示例**:
```json
{
  "entities": [
    {
      "id": "entity_1",
      "name": "人工智能",
      "type": "概念",
      "description": "模拟人类智能的技术",
      "score": 0.95
    },
    {
      "id": "entity_2", 
      "name": "机器学习",
      "type": "技术",
      "description": "AI的一个重要分支",
      "score": 0.88
    }
  ],
  "relationships": [
    {
      "id": "rel_1",
      "source_entity": "entity_1",
      "target_entity": "entity_2", 
      "relation": "包含",
      "description": "人工智能包含机器学习作为其重要分支",
      "score": 0.90
    }
  ],
  "subgraphs": [
    {
      "entities": ["entity_1", "entity_2"],
      "relationships": ["rel_1"],
      "description": "AI与ML的关系子图"
    }
  ]
}
```

## ❌ 错误处理

### 错误响应格式

所有错误响应都遵循统一格式：

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "请求参数验证失败",
    "details": {
      "field": "messages",
      "reason": "messages字段不能为空"
    }
  },
  "request_id": "req_123456789"
}
```

### 常见错误码

| 错误码 | HTTP状态码 | 描述 |
|--------|------------|------|
| `VALIDATION_ERROR` | 400 | 请求参数验证失败 |
| `UNAUTHORIZED` | 401 | 未授权访问 |
| `FORBIDDEN` | 403 | 权限不足 |
| `NOT_FOUND` | 404 | 资源不存在 |
| `RATE_LIMIT_EXCEEDED` | 429 | 请求频率超限 |
| `INTERNAL_ERROR` | 500 | 服务器内部错误 |
| `SERVICE_UNAVAILABLE` | 503 | 服务暂不可用 |

### 错误处理最佳实践

1. **检查HTTP状态码**: 首先检查响应的HTTP状态码
2. **解析错误信息**: 从响应体中获取详细的错误信息
3. **实现重试机制**: 对于临时性错误（如503），实现指数退避重试
4. **记录错误日志**: 记录request_id以便问题追踪
5. **用户友好提示**: 将技术错误转换为用户友好的提示信息

### 示例错误处理代码

```python
import requests
import time
import random

def call_autoflow_api(url, data, max_retries=3):
    for attempt in range(max_retries):
        try:
            response = requests.post(url, json=data)
            
            if response.status_code == 200:
                return response.json()
            elif response.status_code == 429:
                # 速率限制，等待后重试
                wait_time = (2 ** attempt) + random.uniform(0, 1)
                time.sleep(wait_time)
                continue
            elif response.status_code >= 500:
                # 服务器错误，重试
                if attempt < max_retries - 1:
                    time.sleep(2 ** attempt)
                    continue
            
            # 其他错误，不重试
            error_data = response.json()
            raise Exception(f"API错误: {error_data['error']['message']}")
            
        except requests.RequestException as e:
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise Exception(f"网络错误: {str(e)}")
    
    raise Exception("达到最大重试次数")
```
