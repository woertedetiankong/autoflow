# AutoFlow 代码示例和使用场景

## 📋 目录

- [Python SDK示例](#python-sdk示例)
- [REST API示例](#rest-api示例)
- [使用场景](#使用场景)
- [最佳实践](#最佳实践)
- [性能优化](#性能优化)

## 🐍 Python SDK示例

### 基础使用

#### 1. 初始化和配置

```python
import os
from autoflow import Autoflow
from autoflow.configs.db import DatabaseConfig
from autoflow.configs.main import Config
from autoflow.models.llms import LLM
from autoflow.models.embedding_models import EmbeddingModel
from autoflow.configs.knowledge_base import IndexMethod

# 从环境变量加载配置
def create_autoflow():
    return Autoflow.from_config(
        config=Config(
            db=DatabaseConfig(
                host=os.getenv("TIDB_HOST"),
                port=int(os.getenv("TIDB_PORT", "4000")),
                username=os.getenv("TIDB_USERNAME"),
                password=os.getenv("TIDB_PASSWORD"),
                database=os.getenv("TIDB_DATABASE"),
                enable_ssl=os.getenv("TIDB_SSL_ENABLED", "false").lower() == "true",
            )
        )
    )

# 创建AutoFlow实例
af = create_autoflow()
```

#### 2. 创建知识库

```python
from autoflow.chunkers.text import TextChunker
from autoflow.configs.chunkers.text import TextChunkerConfig

# 配置文本分块器
chunker = TextChunker(
    config=TextChunkerConfig(
        chunk_size=512,      # 分块大小
        chunk_overlap=50,    # 重叠大小
        separator="\n\n"     # 分隔符
    )
)

# 创建知识库
kb = af.create_knowledge_base(
    name="企业知识库",
    description="包含公司政策、产品文档和FAQ",
    index_methods=[
        IndexMethod.VECTOR_SEARCH,    # 向量搜索
        IndexMethod.KNOWLEDGE_GRAPH   # 知识图谱
    ],
    llm=LLM("gpt-4o-mini"),
    embedding_model=EmbeddingModel("text-embedding-3-small"),
)

print(f"知识库创建成功: {kb.name}")
```

#### 3. 批量添加文档

```python
import glob
from pathlib import Path

def add_documents_from_directory(kb, directory_path, file_patterns=None):
    """从目录批量添加文档"""
    if file_patterns is None:
        file_patterns = ["*.pdf", "*.md", "*.txt", "*.docx"]
    
    all_files = []
    for pattern in file_patterns:
        all_files.extend(glob.glob(f"{directory_path}/**/{pattern}", recursive=True))
    
    print(f"找到 {len(all_files)} 个文件")
    
    # 批量处理文档
    batch_size = 10
    for i in range(0, len(all_files), batch_size):
        batch_files = all_files[i:i + batch_size]
        print(f"处理批次 {i//batch_size + 1}: {len(batch_files)} 个文件")
        
        try:
            documents = kb.add(batch_files, chunker=chunker)
            print(f"成功添加 {len(documents)} 个文档")
        except Exception as e:
            print(f"批次处理失败: {e}")
            # 逐个处理失败的文件
            for file_path in batch_files:
                try:
                    kb.add(file_path, chunker=chunker)
                    print(f"单独处理成功: {file_path}")
                except Exception as file_error:
                    print(f"文件处理失败 {file_path}: {file_error}")

# 使用示例
add_documents_from_directory(kb, "./documents")
```

#### 4. 智能搜索和问答

```python
def intelligent_search(kb, query, use_kg=True):
    """智能搜索函数"""
    print(f"搜索查询: {query}")
    
    # 1. 向量搜索
    vector_results = kb.search_documents(
        query=query,
        top_k=5,
        similarity_threshold=0.7
    )
    
    print(f"向量搜索找到 {len(vector_results.chunks)} 个相关片段")
    
    # 2. 知识图谱搜索（如果启用）
    if use_kg:
        kg_results = kb.search_knowledge_graph(
            query=query,
            depth=2
        )
        print(f"知识图谱找到 {len(kg_results.relationships)} 个关系")
    
    # 3. 生成答案
    response = kb.ask(query)
    
    return {
        "answer": response.message.content,
        "vector_results": vector_results,
        "kg_results": kg_results if use_kg else None
    }

# 使用示例
result = intelligent_search(kb, "公司的休假政策是什么？")
print(f"答案: {result['answer']}")
```

### 高级功能示例

#### 1. 自定义文档处理器

```python
from autoflow.loaders.base import Loader
from autoflow.data_types import DataType, Document
import json

class JSONLoader(Loader):
    """自定义JSON文档加载器"""
    
    def load(self, source):
        documents = []
        
        if isinstance(source, str):
            source = [source]
        
        for file_path in source:
            with open(file_path, 'r', encoding='utf-8') as f:
                data = json.load(f)
            
            # 将JSON转换为文档
            if isinstance(data, list):
                for i, item in enumerate(data):
                    doc = Document(
                        text=json.dumps(item, ensure_ascii=False, indent=2),
                        metadata={
                            "source": file_path,
                            "index": i,
                            "type": "json_item"
                        }
                    )
                    documents.append(doc)
            else:
                doc = Document(
                    text=json.dumps(data, ensure_ascii=False, indent=2),
                    metadata={
                        "source": file_path,
                        "type": "json_document"
                    }
                )
                documents.append(doc)
        
        return documents

# 使用自定义加载器
json_loader = JSONLoader()
documents = kb.add("./data.json", loader=json_loader)
```

#### 2. 多模态文档处理

```python
from autoflow.loaders.file import FileLoader
from autoflow.data_types import DataType

def process_mixed_documents(kb, file_paths):
    """处理混合类型的文档"""
    results = {}
    
    for file_path in file_paths:
        try:
            # 自动检测文档类型
            data_type = guess_datatype(file_path)
            print(f"处理文件: {file_path}, 类型: {data_type}")
            
            # 根据文件类型选择不同的处理策略
            if data_type == DataType.PDF:
                # PDF文档使用较大的分块
                pdf_chunker = TextChunker(
                    config=TextChunkerConfig(chunk_size=1024, chunk_overlap=100)
                )
                docs = kb.add(file_path, chunker=pdf_chunker)
            elif data_type == DataType.MARKDOWN:
                # Markdown保持结构
                md_chunker = TextChunker(
                    config=TextChunkerConfig(
                        chunk_size=512, 
                        chunk_overlap=50,
                        separator="\n## "  # 按标题分割
                    )
                )
                docs = kb.add(file_path, chunker=md_chunker)
            else:
                # 其他类型使用默认设置
                docs = kb.add(file_path)
            
            results[file_path] = {
                "status": "success",
                "document_count": len(docs),
                "chunk_count": sum(len(doc.chunks) for doc in docs)
            }
            
        except Exception as e:
            results[file_path] = {
                "status": "error",
                "error": str(e)
            }
    
    return results

# 使用示例
mixed_files = [
    "./manual.pdf",
    "./readme.md", 
    "./config.json",
    "./data.csv"
]
results = process_mixed_documents(kb, mixed_files)
```

#### 3. 实时文档监控和更新

```python
import time
import hashlib
from pathlib import Path
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler

class DocumentWatcher(FileSystemEventHandler):
    """文档变化监控器"""
    
    def __init__(self, knowledge_base, watch_directory):
        self.kb = knowledge_base
        self.watch_dir = Path(watch_directory)
        self.file_hashes = {}
        
        # 初始化文件哈希
        self._scan_directory()
    
    def _scan_directory(self):
        """扫描目录并计算文件哈希"""
        for file_path in self.watch_dir.rglob("*"):
            if file_path.is_file() and file_path.suffix in ['.pdf', '.md', '.txt']:
                self.file_hashes[str(file_path)] = self._get_file_hash(file_path)
    
    def _get_file_hash(self, file_path):
        """计算文件哈希值"""
        with open(file_path, 'rb') as f:
            return hashlib.md5(f.read()).hexdigest()
    
    def on_modified(self, event):
        if not event.is_directory:
            self._handle_file_change(event.src_path, "modified")
    
    def on_created(self, event):
        if not event.is_directory:
            self._handle_file_change(event.src_path, "created")
    
    def on_deleted(self, event):
        if not event.is_directory:
            self._handle_file_change(event.src_path, "deleted")
    
    def _handle_file_change(self, file_path, change_type):
        """处理文件变化"""
        try:
            if change_type == "deleted":
                # 从知识库中删除文档
                print(f"文件删除: {file_path}")
                # 这里需要实现删除逻辑
                if file_path in self.file_hashes:
                    del self.file_hashes[file_path]
            
            elif change_type in ["created", "modified"]:
                # 检查文件是否真的发生了变化
                current_hash = self._get_file_hash(file_path)
                old_hash = self.file_hashes.get(file_path)
                
                if current_hash != old_hash:
                    print(f"文件{change_type}: {file_path}")
                    
                    # 重新添加文档到知识库
                    self.kb.add(file_path)
                    self.file_hashes[file_path] = current_hash
                    
                    print(f"文档已更新到知识库: {file_path}")
        
        except Exception as e:
            print(f"处理文件变化失败 {file_path}: {e}")

def start_document_monitoring(kb, watch_directory):
    """启动文档监控"""
    event_handler = DocumentWatcher(kb, watch_directory)
    observer = Observer()
    observer.schedule(event_handler, watch_directory, recursive=True)
    observer.start()
    
    print(f"开始监控目录: {watch_directory}")
    
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
        print("停止监控")
    
    observer.join()

# 使用示例（在单独的进程中运行）
# start_document_monitoring(kb, "./documents")
```

## 🌐 REST API示例

### 使用Python requests

#### 1. 基础聊天API调用

```python
import requests
import json

class AutoFlowClient:
    def __init__(self, base_url, api_key=None):
        self.base_url = base_url.rstrip('/')
        self.session = requests.Session()
        if api_key:
            self.session.headers.update({
                'Authorization': f'Bearer {api_key}'
            })
    
    def chat(self, messages, chat_engine="default", stream=True):
        """发起聊天"""
        url = f"{self.base_url}/api/chats"
        data = {
            "messages": messages,
            "chat_engine": chat_engine,
            "stream": stream
        }
        
        if stream:
            return self._stream_chat(url, data)
        else:
            response = self.session.post(url, json=data)
            response.raise_for_status()
            return response.json()
    
    def _stream_chat(self, url, data):
        """处理流式聊天响应"""
        response = self.session.post(url, json=data, stream=True)
        response.raise_for_status()
        
        for line in response.iter_lines():
            if line:
                line = line.decode('utf-8')
                if line.startswith('data: '):
                    try:
                        event_data = json.loads(line[6:])
                        yield event_data
                    except json.JSONDecodeError:
                        continue
    
    def get_chat_history(self, chat_id):
        """获取聊天历史"""
        url = f"{self.base_url}/api/chats/{chat_id}"
        response = self.session.get(url)
        response.raise_for_status()
        return response.json()

# 使用示例
client = AutoFlowClient("https://your-domain.com", "your-api-key")

# 非流式聊天
messages = [{"role": "user", "content": "什么是机器学习？"}]
result = client.chat(messages, stream=False)
print(f"回答: {result['message']['content']}")

# 流式聊天
print("流式回答:")
for event in client.chat(messages, stream=True):
    if event.get('event_type') == 'text_part':
        print(event['payload'], end='', flush=True)
    elif event.get('event_type') == 'done':
        print("\n[完成]")
        break
```

#### 2. 文档检索API

```python
def search_documents(client, query, knowledge_base_ids=None, top_k=5):
    """搜索文档"""
    url = f"{client.base_url}/api/retrieve/documents"
    data = {
        "query": query,
        "top_k": top_k,
        "similarity_threshold": 0.7
    }
    
    if knowledge_base_ids:
        data["knowledge_base_ids"] = knowledge_base_ids
    
    response = client.session.post(url, json=data)
    response.raise_for_status()
    return response.json()

# 使用示例
search_results = search_documents(
    client, 
    "人工智能的应用领域", 
    knowledge_base_ids=[1, 2],
    top_k=3
)

print(f"找到 {len(search_results['chunks'])} 个相关片段:")
for chunk in search_results['chunks']:
    print(f"- 相关度: {chunk['score']:.3f}")
    print(f"  内容: {chunk['text'][:100]}...")
    print(f"  来源: {chunk['document']['name']}")
```

### 使用JavaScript/Node.js

#### 1. 基础聊天客户端

```javascript
class AutoFlowClient {
    constructor(baseUrl, apiKey = null) {
        this.baseUrl = baseUrl.replace(/\/$/, '');
        this.apiKey = apiKey;
    }

    async chat(messages, options = {}) {
        const {
            chatEngine = 'default',
            stream = true,
            chatId = null
        } = options;

        const url = `${this.baseUrl}/api/chats`;
        const data = {
            messages,
            chat_engine: chatEngine,
            stream,
            ...(chatId && { chat_id: chatId })
        };

        const headers = {
            'Content-Type': 'application/json',
            ...(this.apiKey && { 'Authorization': `Bearer ${this.apiKey}` })
        };

        if (stream) {
            return this._streamChat(url, data, headers);
        } else {
            const response = await fetch(url, {
                method: 'POST',
                headers,
                body: JSON.stringify(data)
            });
            
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }
            
            return await response.json();
        }
    }

    async *_streamChat(url, data, headers) {
        const response = await fetch(url, {
            method: 'POST',
            headers,
            body: JSON.stringify(data)
        });

        if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }

        const reader = response.body.getReader();
        const decoder = new TextDecoder();

        try {
            while (true) {
                const { done, value } = await reader.read();
                if (done) break;

                const chunk = decoder.decode(value);
                const lines = chunk.split('\n');

                for (const line of lines) {
                    if (line.startsWith('data: ')) {
                        try {
                            const eventData = JSON.parse(line.slice(6));
                            yield eventData;
                        } catch (e) {
                            // 忽略解析错误
                        }
                    }
                }
            }
        } finally {
            reader.releaseLock();
        }
    }

    async getChatHistory(chatId) {
        const url = `${this.baseUrl}/api/chats/${chatId}`;
        const headers = {
            ...(this.apiKey && { 'Authorization': `Bearer ${this.apiKey}` })
        };

        const response = await fetch(url, { headers });
        
        if (!response.ok) {
            throw new Error(`HTTP ${response.status}: ${response.statusText}`);
        }
        
        return await response.json();
    }
}

// 使用示例
const client = new AutoFlowClient('https://your-domain.com', 'your-api-key');

// 非流式聊天
async function simpleChat() {
    const messages = [{ role: 'user', content: '什么是深度学习？' }];
    const result = await client.chat(messages, { stream: false });
    console.log('回答:', result.message.content);
}

// 流式聊天
async function streamChat() {
    const messages = [{ role: 'user', content: '解释一下神经网络' }];
    console.log('流式回答:');
    
    for await (const event of client.chat(messages, { stream: true })) {
        if (event.event_type === 'text_part') {
            process.stdout.write(event.payload);
        } else if (event.event_type === 'done') {
            console.log('\n[完成]');
            break;
        }
    }
}

// 运行示例
simpleChat().catch(console.error);
streamChat().catch(console.error);
```

#### 2. 网页集成示例

```html
<!DOCTYPE html>
<html>
<head>
    <title>AutoFlow 聊天示例</title>
    <style>
        .chat-container {
            max-width: 600px;
            margin: 0 auto;
            padding: 20px;
        }
        .message {
            margin: 10px 0;
            padding: 10px;
            border-radius: 5px;
        }
        .user-message {
            background-color: #e3f2fd;
            text-align: right;
        }
        .assistant-message {
            background-color: #f5f5f5;
        }
        .input-container {
            display: flex;
            margin-top: 20px;
        }
        #messageInput {
            flex: 1;
            padding: 10px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }
        #sendButton {
            padding: 10px 20px;
            margin-left: 10px;
            background-color: #2196f3;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
    </style>
</head>
<body>
    <div class="chat-container">
        <h1>AutoFlow 聊天演示</h1>
        <div id="chatMessages"></div>
        <div class="input-container">
            <input type="text" id="messageInput" placeholder="输入您的问题...">
            <button id="sendButton">发送</button>
        </div>
    </div>

    <script>
        class ChatUI {
            constructor(apiBaseUrl, apiKey) {
                this.client = new AutoFlowClient(apiBaseUrl, apiKey);
                this.messagesContainer = document.getElementById('chatMessages');
                this.messageInput = document.getElementById('messageInput');
                this.sendButton = document.getElementById('sendButton');
                
                this.setupEventListeners();
            }

            setupEventListeners() {
                this.sendButton.addEventListener('click', () => this.sendMessage());
                this.messageInput.addEventListener('keypress', (e) => {
                    if (e.key === 'Enter') {
                        this.sendMessage();
                    }
                });
            }

            addMessage(content, isUser = false) {
                const messageDiv = document.createElement('div');
                messageDiv.className = `message ${isUser ? 'user-message' : 'assistant-message'}`;
                messageDiv.textContent = content;
                this.messagesContainer.appendChild(messageDiv);
                this.messagesContainer.scrollTop = this.messagesContainer.scrollHeight;
                return messageDiv;
            }

            async sendMessage() {
                const message = this.messageInput.value.trim();
                if (!message) return;

                // 显示用户消息
                this.addMessage(message, true);
                this.messageInput.value = '';
                this.sendButton.disabled = true;

                // 创建助手消息容器
                const assistantMessageDiv = this.addMessage('', false);
                let assistantMessage = '';

                try {
                    const messages = [{ role: 'user', content: message }];
                    
                    for await (const event of this.client.chat(messages, { stream: true })) {
                        if (event.event_type === 'text_part') {
                            assistantMessage += event.payload;
                            assistantMessageDiv.textContent = assistantMessage;
                        } else if (event.event_type === 'done') {
                            break;
                        }
                    }
                } catch (error) {
                    assistantMessageDiv.textContent = `错误: ${error.message}`;
                    assistantMessageDiv.style.color = 'red';
                } finally {
                    this.sendButton.disabled = false;
                }
            }
        }

        // 初始化聊天界面
        const chatUI = new ChatUI('https://your-domain.com', 'your-api-key');
    </script>
</body>
</html>

## 🎯 使用场景

### 1. 企业知识库问答系统

构建企业内部的智能问答系统，帮助员工快速找到所需信息。

```python
class EnterpriseKnowledgeBase:
    def __init__(self, autoflow_instance):
        self.af = autoflow_instance
        self.departments = {}

    def create_department_kb(self, dept_name, documents_path):
        """为特定部门创建知识库"""
        kb = self.af.create_knowledge_base(
            name=f"{dept_name}知识库",
            description=f"{dept_name}部门的政策、流程和FAQ",
            llm=LLM("gpt-4o-mini"),
            embedding_model=EmbeddingModel("text-embedding-3-small"),
        )

        # 添加部门文档
        kb.add(documents_path)
        self.departments[dept_name] = kb

        return kb

    def smart_routing(self, question):
        """智能路由问题到相关部门"""
        # 简单的关键词路由（实际应用中可以使用更复杂的分类模型）
        routing_keywords = {
            "人事": ["工资", "薪资", "请假", "招聘", "离职", "考勤"],
            "IT": ["系统", "软件", "网络", "密码", "权限", "电脑"],
            "财务": ["报销", "发票", "预算", "成本", "会计"],
            "法务": ["合同", "法律", "合规", "风险", "知识产权"]
        }

        question_lower = question.lower()
        for dept, keywords in routing_keywords.items():
            if any(keyword in question_lower for keyword in keywords):
                if dept in self.departments:
                    return self.departments[dept]

        # 如果没有匹配到特定部门，使用通用知识库
        return self.departments.get("通用", None)

    def answer_question(self, question, user_department=None):
        """回答问题"""
        # 优先使用用户所在部门的知识库
        if user_department and user_department in self.departments:
            primary_kb = self.departments[user_department]
        else:
            primary_kb = self.smart_routing(question)

        if primary_kb:
            response = primary_kb.ask(question)
            return {
                "answer": response.message.content,
                "source_department": primary_kb.name,
                "confidence": "high"
            }
        else:
            return {
                "answer": "抱歉，我无法找到相关信息。请联系相关部门或管理员。",
                "source_department": None,
                "confidence": "low"
            }

# 使用示例
enterprise_kb = EnterpriseKnowledgeBase(af)

# 创建各部门知识库
enterprise_kb.create_department_kb("人事", "./hr_documents/")
enterprise_kb.create_department_kb("IT", "./it_documents/")
enterprise_kb.create_department_kb("财务", "./finance_documents/")

# 回答问题
result = enterprise_kb.answer_question("如何申请年假？", user_department="人事")
print(f"回答: {result['answer']}")
```

### 2. 客户服务聊天机器人

为客户服务构建智能聊天机器人，提供24/7的客户支持。

```python
class CustomerServiceBot:
    def __init__(self, knowledge_base):
        self.kb = knowledge_base
        self.conversation_history = {}
        self.escalation_keywords = [
            "投诉", "不满意", "退款", "取消订单", "人工客服"
        ]

    def should_escalate(self, message):
        """判断是否需要转人工客服"""
        return any(keyword in message for keyword in self.escalation_keywords)

    def get_context_aware_response(self, user_id, message):
        """获取上下文感知的回复"""
        # 获取对话历史
        history = self.conversation_history.get(user_id, [])

        # 构建完整的对话上下文
        context_messages = history + [{"role": "user", "content": message}]

        # 如果需要转人工客服
        if self.should_escalate(message):
            return {
                "response": "我理解您的问题很重要。让我为您转接人工客服，他们会更好地帮助您解决问题。",
                "action": "escalate_to_human",
                "confidence": 1.0
            }

        # 使用知识库回答
        kb_response = self.kb.ask(message)

        # 更新对话历史
        history.append({"role": "user", "content": message})
        history.append({"role": "assistant", "content": kb_response.message.content})

        # 保持历史记录不超过10轮对话
        if len(history) > 20:
            history = history[-20:]

        self.conversation_history[user_id] = history

        return {
            "response": kb_response.message.content,
            "action": "continue_conversation",
            "confidence": 0.8  # 可以根据检索结果的相关度计算
        }

    def handle_customer_query(self, user_id, message):
        """处理客户查询"""
        try:
            result = self.get_context_aware_response(user_id, message)

            # 记录对话日志
            self.log_conversation(user_id, message, result)

            return result

        except Exception as e:
            return {
                "response": "抱歉，系统暂时出现问题。请稍后再试或联系人工客服。",
                "action": "system_error",
                "error": str(e)
            }

    def log_conversation(self, user_id, query, response):
        """记录对话日志（用于分析和改进）"""
        log_entry = {
            "timestamp": time.time(),
            "user_id": user_id,
            "query": query,
            "response": response["response"],
            "action": response["action"],
            "confidence": response.get("confidence", 0)
        }
        # 这里可以写入数据库或日志文件
        print(f"对话日志: {log_entry}")

# 使用示例
customer_bot = CustomerServiceBot(kb)

# 模拟客户对话
user_id = "customer_123"
queries = [
    "你们的退货政策是什么？",
    "我想退货，但是已经超过7天了",
    "这不合理，我要投诉"
]

for query in queries:
    result = customer_bot.handle_customer_query(user_id, query)
    print(f"客户: {query}")
    print(f"机器人: {result['response']}")
    print(f"动作: {result['action']}")
    print("---")
```

### 3. 教育培训助手

为在线教育平台构建智能学习助手。

```python
class EducationAssistant:
    def __init__(self, autoflow_instance):
        self.af = autoflow_instance
        self.course_kbs = {}
        self.student_progress = {}

    def create_course_kb(self, course_id, course_materials):
        """为课程创建知识库"""
        kb = self.af.create_knowledge_base(
            name=f"课程_{course_id}",
            description=f"课程{course_id}的教学材料和参考资料",
            llm=LLM("gpt-4o-mini"),
            embedding_model=EmbeddingModel("text-embedding-3-small"),
        )

        # 添加课程材料
        kb.add(course_materials)
        self.course_kbs[course_id] = kb

        return kb

    def adaptive_learning(self, student_id, course_id, question, difficulty_level="medium"):
        """自适应学习回答"""
        if course_id not in self.course_kbs:
            return "课程不存在"

        kb = self.course_kbs[course_id]

        # 根据难度级别调整回答风格
        difficulty_prompts = {
            "beginner": "请用简单易懂的语言解释，包含具体例子：",
            "medium": "请详细解释概念和原理：",
            "advanced": "请深入分析，包含相关理论和应用："
        }

        enhanced_question = difficulty_prompts.get(difficulty_level, "") + question
        response = kb.ask(enhanced_question)

        # 记录学习进度
        self.update_student_progress(student_id, course_id, question, difficulty_level)

        return response.message.content

    def update_student_progress(self, student_id, course_id, question, difficulty):
        """更新学生学习进度"""
        if student_id not in self.student_progress:
            self.student_progress[student_id] = {}

        if course_id not in self.student_progress[student_id]:
            self.student_progress[student_id][course_id] = {
                "questions_asked": 0,
                "topics_covered": set(),
                "difficulty_distribution": {"beginner": 0, "medium": 0, "advanced": 0}
            }

        progress = self.student_progress[student_id][course_id]
        progress["questions_asked"] += 1
        progress["difficulty_distribution"][difficulty] += 1

        # 简单的主题提取（实际应用中可以使用NLP技术）
        topics = self.extract_topics(question)
        progress["topics_covered"].update(topics)

    def extract_topics(self, question):
        """提取问题中的主题（简化版本）"""
        # 这里可以使用更复杂的NLP技术
        common_topics = {
            "机器学习": ["机器学习", "ML", "算法", "模型"],
            "深度学习": ["深度学习", "神经网络", "DL", "CNN", "RNN"],
            "数据科学": ["数据分析", "统计", "可视化", "pandas"],
            "编程": ["Python", "代码", "函数", "变量"]
        }

        found_topics = []
        question_lower = question.lower()

        for topic, keywords in common_topics.items():
            if any(keyword.lower() in question_lower for keyword in keywords):
                found_topics.append(topic)

        return found_topics

    def generate_study_plan(self, student_id, course_id):
        """生成个性化学习计划"""
        if student_id not in self.student_progress:
            return "暂无学习记录"

        progress = self.student_progress[student_id].get(course_id, {})

        if not progress:
            return "暂无该课程的学习记录"

        # 分析学习情况
        total_questions = progress["questions_asked"]
        topics_covered = len(progress["topics_covered"])
        difficulty_dist = progress["difficulty_distribution"]

        # 生成建议
        suggestions = []

        if difficulty_dist["beginner"] > difficulty_dist["medium"] + difficulty_dist["advanced"]:
            suggestions.append("建议尝试更有挑战性的问题来提升学习深度")

        if topics_covered < 3:
            suggestions.append("建议扩展学习范围，探索更多相关主题")

        if total_questions < 10:
            suggestions.append("建议增加练习频率，多提问多思考")

        return {
            "学习统计": {
                "总提问数": total_questions,
                "涉及主题": list(progress["topics_covered"]),
                "难度分布": difficulty_dist
            },
            "学习建议": suggestions
        }

# 使用示例
edu_assistant = EducationAssistant(af)

# 创建课程知识库
edu_assistant.create_course_kb("ML101", "./ml_course_materials/")

# 学生提问
student_id = "student_456"
course_id = "ML101"

questions = [
    ("什么是机器学习？", "beginner"),
    ("监督学习和无监督学习的区别", "medium"),
    ("梯度下降算法的数学原理", "advanced")
]

for question, difficulty in questions:
    answer = edu_assistant.adaptive_learning(student_id, course_id, question, difficulty)
    print(f"问题 ({difficulty}): {question}")
    print(f"回答: {answer[:200]}...")
    print("---")

# 生成学习计划
study_plan = edu_assistant.generate_study_plan(student_id, course_id)
print("学习计划:", study_plan)
```

## 💡 最佳实践

### 1. 文档预处理

```python
def preprocess_documents(file_paths):
    """文档预处理最佳实践"""
    processed_docs = []

    for file_path in file_paths:
        try:
            # 1. 文件格式验证
            if not is_supported_format(file_path):
                print(f"跳过不支持的文件格式: {file_path}")
                continue

            # 2. 文件大小检查
            file_size = os.path.getsize(file_path)
            if file_size > 50 * 1024 * 1024:  # 50MB
                print(f"文件过大，建议分割: {file_path}")
                continue

            # 3. 文本质量检查
            with open(file_path, 'r', encoding='utf-8') as f:
                content = f.read()

            if len(content.strip()) < 100:
                print(f"文档内容过短，跳过: {file_path}")
                continue

            # 4. 去除噪声内容
            cleaned_content = clean_text(content)

            # 5. 添加元数据
            metadata = extract_metadata(file_path, cleaned_content)

            processed_docs.append({
                "path": file_path,
                "content": cleaned_content,
                "metadata": metadata
            })

        except Exception as e:
            print(f"处理文档失败 {file_path}: {e}")

    return processed_docs

def clean_text(text):
    """清理文本内容"""
    import re

    # 移除多余的空白字符
    text = re.sub(r'\s+', ' ', text)

    # 移除特殊字符（保留基本标点）
    text = re.sub(r'[^\w\s\u4e00-\u9fff.,!?;:()[\]{}"\'-]', '', text)

    # 移除过短的行
    lines = text.split('\n')
    lines = [line.strip() for line in lines if len(line.strip()) > 10]

    return '\n'.join(lines)

def extract_metadata(file_path, content):
    """提取文档元数据"""
    from datetime import datetime
    import hashlib

    return {
        "file_name": os.path.basename(file_path),
        "file_size": os.path.getsize(file_path),
        "content_hash": hashlib.md5(content.encode()).hexdigest(),
        "processed_at": datetime.now().isoformat(),
        "word_count": len(content.split()),
        "char_count": len(content)
    }
```

### 2. 分块策略优化

```python
def optimize_chunking_strategy(documents, kb_type="general"):
    """根据知识库类型优化分块策略"""

    chunking_configs = {
        "technical": TextChunkerConfig(
            chunk_size=1024,
            chunk_overlap=100,
            separator="\n## "  # 按技术文档标题分割
        ),
        "legal": TextChunkerConfig(
            chunk_size=2048,
            chunk_overlap=200,
            separator="\n\n"  # 法律文档需要更大的上下文
        ),
        "faq": TextChunkerConfig(
            chunk_size=512,
            chunk_overlap=50,
            separator="\n\nQ:"  # FAQ按问题分割
        ),
        "general": TextChunkerConfig(
            chunk_size=768,
            chunk_overlap=75,
            separator="\n\n"
        )
    }

    config = chunking_configs.get(kb_type, chunking_configs["general"])
    return TextChunker(config=config)

def adaptive_chunking(document_content, content_type):
    """自适应分块"""
    if "```" in document_content:  # 包含代码块
        return TextChunker(config=TextChunkerConfig(
            chunk_size=1536,
            chunk_overlap=150,
            separator="\n```"
        ))
    elif document_content.count('\n#') > 10:  # Markdown文档
        return TextChunker(config=TextChunkerConfig(
            chunk_size=1024,
            chunk_overlap=100,
            separator="\n# "
        ))
    else:  # 普通文本
        return TextChunker(config=TextChunkerConfig(
            chunk_size=768,
            chunk_overlap=75
        ))
```

### 3. 查询优化

```python
class QueryOptimizer:
    def __init__(self, knowledge_base):
        self.kb = knowledge_base
        self.query_cache = {}

    def optimize_query(self, query):
        """查询优化"""
        # 1. 查询缓存
        query_hash = hashlib.md5(query.encode()).hexdigest()
        if query_hash in self.query_cache:
            return self.query_cache[query_hash]

        # 2. 查询扩展
        expanded_query = self.expand_query(query)

        # 3. 多策略检索
        results = self.multi_strategy_retrieval(expanded_query)

        # 4. 结果缓存
        self.query_cache[query_hash] = results

        return results

    def expand_query(self, query):
        """查询扩展"""
        # 添加同义词和相关词
        synonyms = {
            "AI": ["人工智能", "机器智能"],
            "ML": ["机器学习", "机器学习算法"],
            "DL": ["深度学习", "神经网络"]
        }

        expanded_terms = [query]
        for term, syns in synonyms.items():
            if term.lower() in query.lower():
                expanded_terms.extend(syns)

        return " ".join(expanded_terms)

    def multi_strategy_retrieval(self, query):
        """多策略检索"""
        # 1. 向量检索
        vector_results = self.kb.search_documents(
            query=query,
            top_k=10,
            similarity_threshold=0.6
        )

        # 2. 知识图谱检索
        kg_results = self.kb.search_knowledge_graph(
            query=query,
            depth=2
        )

        # 3. 结果融合
        return self.fuse_results(vector_results, kg_results)

    def fuse_results(self, vector_results, kg_results):
        """结果融合"""
        # 简单的分数加权融合
        fused_results = []

        for chunk in vector_results.chunks:
            score = chunk.score * 0.7  # 向量搜索权重

            # 检查是否在知识图谱中有相关实体
            for entity in kg_results.entities:
                if entity.name.lower() in chunk.text.lower():
                    score += 0.3  # 知识图谱加权
                    break

            fused_results.append({
                "chunk": chunk,
                "fused_score": score
            })

        # 按融合分数排序
        fused_results.sort(key=lambda x: x["fused_score"], reverse=True)

        return fused_results[:5]  # 返回top5
```

## ⚡ 性能优化

### 1. 批处理优化

```python
class BatchProcessor:
    def __init__(self, knowledge_base, batch_size=10):
        self.kb = knowledge_base
        self.batch_size = batch_size

    def batch_add_documents(self, file_paths):
        """批量添加文档"""
        total_files = len(file_paths)
        processed = 0

        for i in range(0, total_files, self.batch_size):
            batch = file_paths[i:i + self.batch_size]

            try:
                # 并行处理批次
                with ThreadPoolExecutor(max_workers=4) as executor:
                    futures = [
                        executor.submit(self.process_single_file, file_path)
                        for file_path in batch
                    ]

                    for future in as_completed(futures):
                        try:
                            result = future.result()
                            processed += 1
                            print(f"进度: {processed}/{total_files}")
                        except Exception as e:
                            print(f"处理失败: {e}")

            except Exception as e:
                print(f"批次处理失败: {e}")

    def process_single_file(self, file_path):
        """处理单个文件"""
        return self.kb.add(file_path)

# 使用示例
batch_processor = BatchProcessor(kb, batch_size=5)
batch_processor.batch_add_documents(large_file_list)
```

### 2. 缓存策略

```python
import redis
import pickle
from functools import wraps

class CacheManager:
    def __init__(self, redis_url="redis://localhost:6379"):
        self.redis_client = redis.from_url(redis_url)
        self.default_ttl = 3600  # 1小时

    def cache_result(self, ttl=None):
        """结果缓存装饰器"""
        def decorator(func):
            @wraps(func)
            def wrapper(*args, **kwargs):
                # 生成缓存键
                cache_key = self.generate_cache_key(func.__name__, args, kwargs)

                # 尝试从缓存获取
                cached_result = self.get_cached_result(cache_key)
                if cached_result is not None:
                    return cached_result

                # 执行函数并缓存结果
                result = func(*args, **kwargs)
                self.cache_result_data(cache_key, result, ttl or self.default_ttl)

                return result
            return wrapper
        return decorator

    def generate_cache_key(self, func_name, args, kwargs):
        """生成缓存键"""
        import hashlib
        key_data = f"{func_name}:{str(args)}:{str(sorted(kwargs.items()))}"
        return hashlib.md5(key_data.encode()).hexdigest()

    def get_cached_result(self, cache_key):
        """获取缓存结果"""
        try:
            cached_data = self.redis_client.get(cache_key)
            if cached_data:
                return pickle.loads(cached_data)
        except Exception as e:
            print(f"缓存读取失败: {e}")
        return None

    def cache_result_data(self, cache_key, data, ttl):
        """缓存结果数据"""
        try:
            serialized_data = pickle.dumps(data)
            self.redis_client.setex(cache_key, ttl, serialized_data)
        except Exception as e:
            print(f"缓存写入失败: {e}")

# 使用示例
cache_manager = CacheManager()

class OptimizedKnowledgeBase:
    def __init__(self, kb):
        self.kb = kb

    @cache_manager.cache_result(ttl=1800)  # 缓存30分钟
    def cached_search(self, query, top_k=5):
        """带缓存的搜索"""
        return self.kb.search_documents(query=query, top_k=top_k)

    @cache_manager.cache_result(ttl=3600)  # 缓存1小时
    def cached_kg_search(self, query, depth=2):
        """带缓存的知识图谱搜索"""
        return self.kb.search_knowledge_graph(query=query, depth=depth)

# 使用优化后的知识库
optimized_kb = OptimizedKnowledgeBase(kb)
```

### 3. 异步处理

```python
import asyncio
import aiohttp
from concurrent.futures import ThreadPoolExecutor

class AsyncAutoFlowClient:
    def __init__(self, base_url, api_key=None):
        self.base_url = base_url
        self.api_key = api_key
        self.session = None

    async def __aenter__(self):
        headers = {}
        if self.api_key:
            headers['Authorization'] = f'Bearer {self.api_key}'

        self.session = aiohttp.ClientSession(headers=headers)
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if self.session:
            await self.session.close()

    async def async_chat(self, messages, chat_engine="default"):
        """异步聊天"""
        url = f"{self.base_url}/api/chats"
        data = {
            "messages": messages,
            "chat_engine": chat_engine,
            "stream": False
        }

        async with self.session.post(url, json=data) as response:
            response.raise_for_status()
            return await response.json()

    async def batch_chat(self, message_batches):
        """批量异步聊天"""
        tasks = [
            self.async_chat(messages)
            for messages in message_batches
        ]

        results = await asyncio.gather(*tasks, return_exceptions=True)
        return results

# 使用示例
async def main():
    async with AsyncAutoFlowClient("https://your-domain.com", "your-api-key") as client:
        # 批量处理多个查询
        queries = [
            [{"role": "user", "content": "什么是机器学习？"}],
            [{"role": "user", "content": "深度学习的应用"}],
            [{"role": "user", "content": "AI的发展历史"}]
        ]

        results = await client.batch_chat(queries)

        for i, result in enumerate(results):
            if isinstance(result, Exception):
                print(f"查询 {i+1} 失败: {result}")
            else:
                print(f"查询 {i+1} 结果: {result['message']['content'][:100]}...")

# 运行异步示例
# asyncio.run(main())
```
```
