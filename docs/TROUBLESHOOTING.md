# AutoFlow 故障排除指南

## 📋 目录

- [安装问题](#安装问题)
- [配置问题](#配置问题)
- [运行时错误](#运行时错误)
- [性能问题](#性能问题)
- [API错误](#api错误)
- [数据库问题](#数据库问题)
- [常见问题FAQ](#常见问题faq)

## 🔧 安装问题

### 1. Docker Compose启动失败

**问题**: `docker-compose up -d` 失败

**可能原因和解决方案**:

```bash
# 检查Docker版本
docker --version
docker-compose --version

# 确保版本满足要求
# Docker: 20.10+
# Docker Compose: 2.0+

# 如果版本过低，更新Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**端口冲突问题**:
```bash
# 检查端口占用
sudo netstat -tulpn | grep :3000
sudo netstat -tulpn | grep :8000

# 修改docker-compose.yml中的端口映射
ports:
  - "3001:3000"  # 前端端口改为3001
  - "8001:80"    # 后端端口改为8001
```

### 2. Python SDK安装失败

**问题**: `pip install autoflow-ai` 失败

**解决方案**:
```bash
# 检查Python版本
python --version  # 需要3.10+

# 升级pip
pip install --upgrade pip

# 使用清华源安装
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple autoflow-ai

# 如果仍然失败，从源码安装
git clone https://github.com/pingcap/autoflow.git
cd autoflow/core
pip install -e .
```

### 3. 依赖冲突

**问题**: 包依赖冲突

**解决方案**:
```bash
# 创建虚拟环境
python -m venv autoflow-env
source autoflow-env/bin/activate  # Linux/Mac
# 或
autoflow-env\Scripts\activate  # Windows

# 安装依赖
pip install autoflow-ai

# 检查依赖冲突
pip check
```

## ⚙️ 配置问题

### 1. 数据库连接失败

**错误信息**: `Connection refused` 或 `Access denied`

**检查清单**:
```bash
# 1. 检查数据库服务状态
mysql -h your-host -P 4000 -u your-username -p

# 2. 验证环境变量
echo $TIDB_HOST
echo $TIDB_PORT
echo $TIDB_USERNAME

# 3. 检查防火墙设置
telnet your-tidb-host 4000

# 4. 验证SSL配置
# 如果使用TiDB Cloud，确保启用SSL
TIDB_SSL_ENABLED=true
```

**TiDB Cloud连接示例**:
```bash
# .env文件配置
TIDB_HOST=gateway01.us-west-2.prod.aws.tidbcloud.com
TIDB_PORT=4000
TIDB_USERNAME=your-username
TIDB_PASSWORD=your-password
TIDB_DATABASE=test
TIDB_SSL_ENABLED=true
```

### 2. OpenAI API配置错误

**错误信息**: `Invalid API key` 或 `Rate limit exceeded`

**解决方案**:
```bash
# 1. 验证API密钥
curl -H "Authorization: Bearer $OPENAI_API_KEY" \
     https://api.openai.com/v1/models

# 2. 检查配额和限制
# 登录OpenAI控制台查看使用情况

# 3. 使用代理或其他提供商
OPENAI_BASE_URL=https://your-proxy-url/v1
# 或使用Azure OpenAI
OPENAI_API_TYPE=azure
AZURE_OPENAI_ENDPOINT=https://your-resource.openai.azure.com/
```

### 3. 环境变量未生效

**问题**: 配置的环境变量没有被读取

**解决方案**:
```bash
# 1. 检查.env文件位置
ls -la .env

# 2. 验证文件格式（无BOM，Unix换行符）
file .env
cat -A .env

# 3. 重启服务
docker-compose down
docker-compose up -d

# 4. 手动加载环境变量
source .env
export $(cat .env | xargs)
```

## 🚨 运行时错误

### 1. 内存不足错误

**错误信息**: `Out of memory` 或 `Killed`

**解决方案**:
```bash
# 1. 检查系统内存
free -h
top

# 2. 调整Docker内存限制
# 在docker-compose.yml中添加
services:
  backend:
    deploy:
      resources:
        limits:
          memory: 4G
        reservations:
          memory: 2G

# 3. 优化批处理大小
# 在代码中减少batch_size
batch_size = 5  # 从10减少到5
```

### 2. 文档处理失败

**错误信息**: `Failed to process document` 或 `Unsupported format`

**诊断步骤**:
```python
# 1. 检查文件格式和大小
import os
file_path = "problematic_file.pdf"
print(f"文件大小: {os.path.getsize(file_path) / 1024 / 1024:.2f} MB")
print(f"文件扩展名: {os.path.splitext(file_path)[1]}")

# 2. 测试文件读取
try:
    with open(file_path, 'rb') as f:
        content = f.read(1024)  # 读取前1KB
    print("文件可读")
except Exception as e:
    print(f"文件读取失败: {e}")

# 3. 使用更小的分块大小
chunker = TextChunker(
    config=TextChunkerConfig(
        chunk_size=256,  # 减小分块大小
        chunk_overlap=25
    )
)
```

### 3. 知识图谱提取失败

**错误信息**: `Knowledge graph extraction failed`

**解决方案**:
```python
# 1. 检查文本质量
def check_text_quality(text):
    if len(text.strip()) < 50:
        return False, "文本过短"
    
    # 检查是否包含有意义的内容
    words = text.split()
    if len(words) < 10:
        return False, "词汇量不足"
    
    return True, "文本质量良好"

# 2. 预处理文本
def preprocess_for_kg(text):
    # 移除过多的换行符
    text = re.sub(r'\n+', '\n', text)
    
    # 移除特殊字符
    text = re.sub(r'[^\w\s\u4e00-\u9fff.,!?;:()[\]{}"\'-]', '', text)
    
    return text.strip()

# 3. 分段处理
def process_in_segments(kb, long_text, segment_size=2000):
    segments = [long_text[i:i+segment_size] 
                for i in range(0, len(long_text), segment_size)]
    
    for i, segment in enumerate(segments):
        try:
            kb.add_text(segment)
            print(f"段落 {i+1} 处理成功")
        except Exception as e:
            print(f"段落 {i+1} 处理失败: {e}")
```

## 🐌 性能问题

### 1. 查询响应慢

**问题**: 搜索或问答响应时间过长

**优化方案**:
```python
# 1. 启用查询缓存
from functools import lru_cache

@lru_cache(maxsize=100)
def cached_search(query, top_k=5):
    return kb.search_documents(query=query, top_k=top_k)

# 2. 调整检索参数
result = kb.search_documents(
    query=query,
    top_k=3,  # 减少返回数量
    similarity_threshold=0.8  # 提高阈值
)

# 3. 使用异步处理
import asyncio

async def async_search(queries):
    tasks = [search_single_query(q) for q in queries]
    return await asyncio.gather(*tasks)
```

### 2. 内存使用过高

**监控和优化**:
```python
import psutil
import gc

def monitor_memory():
    process = psutil.Process()
    memory_info = process.memory_info()
    print(f"内存使用: {memory_info.rss / 1024 / 1024:.2f} MB")

# 定期清理内存
def cleanup_memory():
    gc.collect()
    
# 批处理优化
def process_documents_in_batches(file_paths, batch_size=5):
    for i in range(0, len(file_paths), batch_size):
        batch = file_paths[i:i + batch_size]
        
        # 处理批次
        for file_path in batch:
            kb.add(file_path)
        
        # 清理内存
        cleanup_memory()
        monitor_memory()
```

### 3. 数据库查询慢

**优化建议**:
```sql
-- 1. 检查索引
SHOW INDEX FROM your_table;

-- 2. 分析查询计划
EXPLAIN SELECT * FROM chunks WHERE embedding <-> '[...]' < 0.8;

-- 3. 优化向量索引
-- 确保向量列有适当的索引
ALTER TABLE chunks ADD VECTOR INDEX idx_embedding (embedding);

-- 4. 定期更新统计信息
ANALYZE TABLE chunks;
```

## 🌐 API错误

### 1. 认证失败

**错误码**: 401 Unauthorized

**解决方案**:
```python
# 1. 检查API密钥格式
api_key = "your-api-key"
if not api_key.startswith("sk-"):
    print("API密钥格式可能不正确")

# 2. 验证请求头
headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type": "application/json"
}

# 3. 测试认证
import requests
response = requests.get(
    "https://your-domain.com/api/healthz",
    headers=headers
)
print(f"认证状态: {response.status_code}")
```

### 2. 请求频率限制

**错误码**: 429 Too Many Requests

**解决方案**:
```python
import time
import random
from functools import wraps

def rate_limit_retry(max_retries=3, base_delay=1):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(*args, **kwargs)
                except requests.exceptions.HTTPError as e:
                    if e.response.status_code == 429:
                        if attempt < max_retries - 1:
                            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
                            print(f"请求限制，等待 {delay:.2f} 秒后重试...")
                            time.sleep(delay)
                            continue
                    raise
            return None
        return wrapper
    return decorator

@rate_limit_retry(max_retries=3)
def api_call():
    # 你的API调用代码
    pass
```

### 3. 流式响应中断

**问题**: 流式聊天响应意外中断

**解决方案**:
```python
def robust_stream_chat(client, messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            response_parts = []
            
            for event in client.chat(messages, stream=True):
                if event.get('event_type') == 'text_part':
                    response_parts.append(event['payload'])
                elif event.get('event_type') == 'error_part':
                    raise Exception(f"流式响应错误: {event['payload']}")
                elif event.get('event_type') == 'done':
                    return ''.join(response_parts)
            
        except Exception as e:
            print(f"尝试 {attempt + 1} 失败: {e}")
            if attempt < max_retries - 1:
                time.sleep(2 ** attempt)
                continue
            raise
    
    return None
```

## 🗄️ 数据库问题

### 1. 连接池耗尽

**错误信息**: `Connection pool exhausted`

**解决方案**:
```python
# 1. 调整连接池配置
from sqlalchemy import create_engine

engine = create_engine(
    database_url,
    pool_size=20,          # 增加连接池大小
    max_overflow=30,       # 允许的溢出连接数
    pool_timeout=30,       # 连接超时时间
    pool_recycle=3600,     # 连接回收时间
    pool_pre_ping=True     # 连接前ping测试
)

# 2. 确保连接正确关闭
from contextlib import contextmanager

@contextmanager
def get_db_session():
    session = Session(engine)
    try:
        yield session
        session.commit()
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()

# 使用方式
with get_db_session() as session:
    # 数据库操作
    pass
```

### 2. 向量索引性能问题

**问题**: 向量搜索很慢

**优化方案**:
```sql
-- 1. 检查向量索引状态
SHOW CREATE TABLE chunks;

-- 2. 重建向量索引
ALTER TABLE chunks DROP INDEX idx_embedding;
ALTER TABLE chunks ADD VECTOR INDEX idx_embedding (embedding);

-- 3. 调整向量搜索参数
-- 在应用中调整相似度阈值
similarity_threshold = 0.8  # 提高阈值减少搜索范围
top_k = 5  # 减少返回数量
```

### 3. 数据一致性问题

**问题**: 文档和向量数据不一致

**检查和修复**:
```python
def check_data_consistency(kb):
    """检查数据一致性"""
    # 检查文档数量
    doc_count = session.query(Document).count()
    
    # 检查分块数量
    chunk_count = session.query(Chunk).count()
    
    # 检查向量数量
    vector_count = session.execute(
        "SELECT COUNT(*) FROM chunks WHERE embedding IS NOT NULL"
    ).scalar()
    
    print(f"文档数量: {doc_count}")
    print(f"分块数量: {chunk_count}")
    print(f"向量数量: {vector_count}")
    
    if chunk_count != vector_count:
        print("⚠️ 警告: 分块和向量数量不一致")
        return False
    
    return True

def repair_missing_vectors(kb):
    """修复缺失的向量"""
    chunks_without_vectors = session.query(Chunk).filter(
        Chunk.embedding.is_(None)
    ).all()
    
    print(f"发现 {len(chunks_without_vectors)} 个缺失向量的分块")
    
    for chunk in chunks_without_vectors:
        try:
            # 重新生成向量
            embedding = kb._embedding_model.embed_text(chunk.text)
            chunk.embedding = embedding
            session.commit()
            print(f"修复分块 {chunk.id}")
        except Exception as e:
            print(f"修复分块 {chunk.id} 失败: {e}")
```

## ❓ 常见问题FAQ

### Q1: 如何选择合适的分块大小？

**A**: 分块大小选择指南：
- **技术文档**: 1024-1536 tokens（保持代码完整性）
- **法律文档**: 2048+ tokens（需要更多上下文）
- **FAQ文档**: 512 tokens（问答对相对独立）
- **一般文档**: 768-1024 tokens（平衡性能和质量）

### Q2: 知识图谱和向量搜索哪个更好？

**A**: 两者各有优势，建议结合使用：
- **向量搜索**: 语义相似度高，适合模糊查询
- **知识图谱**: 关系明确，适合精确查询
- **混合检索**: 结合两者优势，提供最佳结果

### Q3: 如何提高问答质量？

**A**: 质量提升策略：
1. **文档质量**: 确保源文档准确、完整
2. **分块策略**: 保持语义完整性
3. **模型选择**: 使用更强的LLM模型
4. **提示优化**: 优化系统提示词
5. **结果过滤**: 设置合适的相似度阈值

### Q4: 支持哪些文档格式？

**A**: 当前支持的格式：
- **文本**: .txt, .md, .csv
- **文档**: .pdf, .docx, .doc
- **网页**: .html, .htm
- **代码**: .py, .js, .java, .cpp
- **数据**: .json, .xml, .yaml

### Q5: 如何监控系统性能？

**A**: 监控建议：
```python
# 1. 响应时间监控
import time

def monitor_response_time(func):
    def wrapper(*args, **kwargs):
        start_time = time.time()
        result = func(*args, **kwargs)
        end_time = time.time()
        print(f"响应时间: {end_time - start_time:.2f}秒")
        return result
    return wrapper

# 2. 内存使用监控
import psutil

def log_system_stats():
    cpu_percent = psutil.cpu_percent()
    memory = psutil.virtual_memory()
    print(f"CPU: {cpu_percent}%, 内存: {memory.percent}%")

# 3. 数据库连接监控
def check_db_connections():
    result = session.execute("SHOW PROCESSLIST")
    connection_count = len(result.fetchall())
    print(f"数据库连接数: {connection_count}")
```

---

## 🆘 获取帮助

如果以上解决方案都无法解决您的问题，请：

1. **查看日志**: 检查详细的错误日志
2. **搜索Issues**: 在GitHub仓库中搜索相似问题
3. **提交Issue**: 提供详细的错误信息和复现步骤
4. **社区讨论**: 在Discord或讨论区寻求帮助

**提交Issue时请包含**:
- 错误信息和堆栈跟踪
- 系统环境信息
- 复现步骤
- 相关配置文件（隐藏敏感信息）
