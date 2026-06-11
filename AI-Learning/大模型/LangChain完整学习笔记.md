
## 目录

- [[#一、LangChain 概述]]
- [[#二、Model I/O —— 模型输入输出]]
- [[#三、Chains —— 链式调用（LCEL）]]
- [[#四、Retrieval —— RAG 检索增强生成]]
- [[#五、Agents —— 智能体]]
- [[#六、MCP —— 模型上下文协议]]
- [[#七、面试重点速查]]

---

## 一、LangChain 概述

### 1.1 什么是 LangChain

> LangChain 是 2022 年 10 月由哈佛大学 **Harrison Chase** 发起的开源框架，用于开发由大语言模型（LLMs）驱动的应用程序。

**核心价值：**
- **统一接口**：不同模型（GPT、Qwen、DeepSeek）统一调用方式，切换零成本
- **简化开发**：封装底层复杂的入出参构建，开发者专注业务逻辑
- **现成的 Agent 构建方法**：结构化、易组合、易扩展

**相关链接：**
- GitHub：https://github.com/langchain-ai/langchain
- 官方文档：https://docs.langchain.com

---

### 1.2 包结构总览

| 包 | 核心职责 |
|---|---|
| `langchain` | 主入口，包含所有实现 |
| `langchain-core` | 核心接口和抽象（Runnable、BaseMessage等） |
| `langchain-openai` | OpenAI / DeepSeek 集成 |
| `langchain-mcp-adapters` | MCP 工具接入 |
| `langchain-text-splitters` | 文本切分工具 |

---

### 1.3 四大核心模块

```
┌─────────────────────────────────────────────────────┐
│                   LangChain 核心模块                  │
├──────────────┬──────────────┬──────────────┬─────────┤
│  Model I/O   │    Chains    │  Retrieval   │ Agents  │
│  ──────────  │  ──────────  │  ──────────  │ ─────── │
│  提示模板     │  组件串联     │  RAG检索      │ 自主决策 │
│  模型调用     │  LCEL管道    │  向量数据库    │ 工具调用 │
│  输出解析     │  并行/分支   │  文档加载     │ 记忆管理 │
└──────────────┴──────────────┴──────────────┴─────────┘
```

---

## 二、Model I/O —— 模型输入输出

### 2.1 整体流程

```
用户输入 → [Prompt Template] → [Model] → [Output Parser] → 结构化结果
           格式化输入          调用LLM     解析输出
```

---

### 2.2 调用在线模型

#### 方式一：OpenAI SDK（ChatCompletions API）

```python
# pip install openai python-dotenv
from openai import OpenAI
import os
from dotenv import load_dotenv

load_dotenv()  # 加载 .env 文件

client = OpenAI(
    base_url=os.getenv("OPENAI_BASE_URL"),
    api_key=os.getenv("OPENAI_API_KEY"),
)

completion = client.chat.completions.create(
    model="gpt-4o-mini",
    messages=[{"role": "user", "content": "将'你好'翻译成意大利语"}],
)
print(completion.choices[0].message.content)
```

> ⚠️ **安全提示**：API_KEY 必须通过环境变量管理，绝对不能硬编码，且不要将 `.env` 放入 git 管理！

#### 方式二：LangChain 统一调用

```python
# pip install langchain langchain-openai
from langchain.chat_models import init_chat_model

# 方式A：通用初始化
llm = init_chat_model(
    model="gpt-4o-mini",
    model_provider="openai",
    base_url=os.getenv("OPENAI_BASE_URL"),
    api_key=os.getenv("OPENAI_API_KEY"),
)

# 方式B：特定包的类
from langchain_openai import ChatOpenAI
llm = ChatOpenAI(model="gpt-4o-mini", temperature=0.0)

resp = llm.invoke("你好")
print(resp.content)  # resp 是 AIMessage 对象
```

**`init_chat_model` 关键参数：**

| 参数 | 说明 |
|---|---|
| `model` | 模型名称，如 `gpt-4o-mini` |
| `model_provider` | 厂商，如 `openai`, `deepseek` |
| `temperature` | 随机性，0=确定，1=创意，生产环境常设 0 |
| `max_tokens` | 限制输出长度（1个中文Token ≈ 1-1.8个汉字） |
| `max_retries` | 失败重试次数 |

---

### 2.3 消息类型（Message Types）

> 💡 这是面试高频考点！

```python
from langchain_core.messages import HumanMessage, SystemMessage, AIMessage, ToolMessage

messages = [
    SystemMessage(content="你是一个专业的数学助手"),   # 系统指令
    HumanMessage(content="1+1等于几"),               # 用户输入
    AIMessage(content="1+1=2"),                     # 模型历史回复
    # ToolMessage 仅在工具调用场景使用
]
```

**三种等价写法：**
```python
# 写法1：Message 对象（推荐）
[SystemMessage("你是助手"), HumanMessage("你好")]

# 写法2：元组
[("system", "你是助手"), ("user", "你好")]

# 写法3：OpenAI 原生 dict
[{"role": "system", "content": "你是助手"}, {"role": "user", "content": "你好"}]
```

---

### 2.4 四种调用方式

```python
# 1. 同步调用（基础）
resp = llm.invoke(messages)

# 2. 异步调用（生产推荐，提升并发性能）
import asyncio
async def call():
    resp = await llm.ainvoke(messages)
asyncio.run(call())

# 3. 流式调用（打字机效果）
for chunk in llm.stream(messages):
    print(chunk.content, end="")

# 4. 批次调用（并行处理多个请求）
responses = llm.batch([messages1, messages2, messages3])
```

---

### 2.5 调用本地模型（Ollama）

> Ollama：本地运行大模型的集成框架，适合原型开发和本地调试

```bash
# 安装并运行模型
ollama run qwen3:8b
```

```python
# pip install langchain-ollama
from langchain_ollama import ChatOllama

llm = ChatOllama(
    model="qwen3:8b",
    base_url="http://localhost:11434",  # 非默认端口时需指定
)
resp = llm.invoke({"role": "user", "content": "你好"})
print(resp.content)
```

> ⚠️ 注意：Ollama 适合本地开发，生产环境应使用 **vLLM** 等部署框架

---

### 2.6 结构化输出（Output Parser）

#### 流程图

```
用户提问
   ↓
定义 Pydantic Schema（数据结构）
   ↓
llm.with_structured_output(schema) → 新的 llm 对象
   ↓
调用新 llm → 直接返回 Pydantic 对象
```

#### 核心代码

```python
from pydantic import BaseModel, Field
from langchain_openai import ChatOpenAI

# 1. 定义数据结构
class CalendarEvent(BaseModel):
    name: str
    date: str
    participants: list[str]

# 2. 创建结构化输出 LLM
llm = ChatOpenAI(model="gpt-4o-mini")
structured_llm = llm.with_structured_output(schema=CalendarEvent)

# 3. 调用 → 直接返回 Pydantic 对象
result = structured_llm.invoke("Alice 和 Bob 周五去科学展")
print(result.name)          # 字符串
print(result.participants)  # 列表
print(type(result))         # <class 'CalendarEvent'>
```

> 💡 **关键优势**：不管换什么模型（OpenAI / Gemini / DeepSeek），调用方式完全相同，只需修改 llm 实例化代码！

#### 其他 Output Parser

| Parser | 用途 |
|---|---|
| `StrOutputParser` | 提取纯文本字符串 |
| `JsonOutputParser` | 输出 JSON（依赖 Prompt 约束） |
| `PydanticOutputParser` | 基于 Pydantic 模型解析 |
| `CommaSeparatedListOutputParser` | 逗号分隔列表 |
| `XMLOutputParser` | XML 格式 |

---

### 2.7 提示词模板（Prompt Template）

```python
from langchain_core.prompts import ChatPromptTemplate

# 创建带变量的模板
template = ChatPromptTemplate.from_messages([
    ("system", "你是一个专业的评论员"),
    ("human", "请评价{product}的优缺点，包括{aspect1}和{aspect2}。"),
])

# 填入变量，生成消息列表
messages = template.invoke({
    "product": "iPhone 15",
    "aspect1": "性能",
    "aspect2": "外观"
})

# 传给 LLM
resp = llm.invoke(messages)
```

---

## 三、Chains —— 链式调用（LCEL）

### 3.1 Runnable 接口

> 💡 这是 LangChain 的设计精髓！

LangChain 所有核心组件（LLM、Prompt、Parser、Retriever…）都实现了 **Runnable 接口**，统一暴露：

| 方法 | 作用 |
|---|---|
| `invoke / ainvoke` | 单个输入 → 单个输出 |
| `batch / abatch` | 多个输入 → 多个输出（并行） |
| `stream / astream` | 单个输入 → 流式输出 |

**没有统一接口时的痛苦：**
```python
# ❌ 每个组件用法不同，组合时需手动适配
prompt_text = prompt.format(topic="猫")
model_out = model.generate(prompt_text)
result = parser.parse(model_out)
```

**有了 Runnable 之后：**
```python
# ✅ 所有组件统一 invoke
prompt_text = prompt.invoke({"topic": "猫"})
model_out = model.invoke(prompt_text)
result = parser.invoke(model_out)
```

---

### 3.2 LCEL 管道语法

> **LCEL（LangChain Expression Language）**：用 `|` 管道符组合 Runnable，自动处理类型匹配和中间结果传递

```python
# ✨ LCEL 核心语法
chain = prompt | llm | parser

# 一行代码替代三行，且可以直接 stream/batch
result = chain.invoke({"topic": "猫"})
```

**内部等价：**
```python
chain = prompt | llm | parser
# 等同于
chain = RunnableSequence([prompt, llm, parser])
```

---

### 3.3 串行：RunnableSequence

```
输入 → [Prompt] → [LLM] → [Parser] → 输出
         ↓          ↓         ↓
       消息列表    AIMessage   字符串
```

```python
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
from langchain.chat_models import init_chat_model

llm = init_chat_model(model="gpt-4o-mini", model_provider="openai")
chain = PromptTemplate.from_template("讲一个关于{topic}的笑话") | llm | StrOutputParser()

result = chain.invoke({"topic": "人工智能"})
print(result)  # 直接是字符串
```

---

### 3.4 并行：RunnableParallel

```
              ┌──→ [english_chain] ──→ 英文翻译
输入 {"topic"}─┤
              └──→ [korean_chain]  ──→ 韩文翻译
                         ↓
              {"english": "...", "korean": "..."}
```

```python
from langchain_core.runnables import RunnableParallel

# 方式1：显式使用 RunnableParallel
map_chain = RunnableParallel(english=english_chain, korean=korean_chain)

# 方式2：LCEL 字典语法（推荐，更简洁）
map_chain = {
    "english": english_chain,
    "korean": korean_chain,
} | summary_chain  # 结果传给下一个 chain
```

**实战案例：多模型并行对比，再由第三个模型总结**

```python
# 并行用两个模型赏析同一首诗，再用 deepseek 对比总结
chain = {
    "paragraph_1": prompt | gpt_llm | StrOutputParser(),
    "paragraph_2": prompt | gemini_llm | StrOutputParser(),
} | summary_prompt | deepseek_llm | StrOutputParser()
```

---

### 3.5 其他 Runnable 组件

| 组件 | 用途 |
|---|---|
| `RunnableLambda` | 将普通函数封装成 Runnable |
| `RunnableBranch` | if-else 路由到不同分支 |
| `RunnablePassthrough` | 透传输入，用于在管道中保留原始数据 |
| `RunnableWithFallbacks` | 失败兜底，自动切换备用 Runnable |

> 💡 随着 Agent/LangGraph 的普及，LCEL Chain 的重要性降低，**了解原理即可**，核心精力放在 Agent。

---

## 四、Retrieval —— RAG 检索增强生成

### 4.1 为什么需要 RAG？

```
大模型的三大局限：
┌──────────────┬────────────────────────────────────────┐
│ 知识滞后      │ 训练数据有截止日期，无法获取最新信息        │
│ 专有知识缺失  │ 企业内部/垂直领域知识无法学全              │
│ 幻觉         │ 编造不存在的事实，在专业领域尤其危险        │
└──────────────┴────────────────────────────────────────┘

RAG 的解决方案：给大模型一本"参考书"
→ 检索相关文档片段 + 放入 Prompt → 基于事实生成答案
```

**RAG 的优缺点：**

| | 优点 | 缺点 |
|---|---|---|
| vs 提示词工程 | 更丰富的上下文，无需用户描述大量背景 | - |
| vs 模型微调 | 时效性强、成本低、保护数据隐私 | 响应延迟更高，消耗更多 Token |

---

### 4.2 RAG 完整流程图

```
【索引阶段（离线）】
原始文档（PDF/Word/MD）
   ↓ 文档加载器（Loader）
Document 对象
   ↓ 文档切分器（TextSplitter）
Chunk 列表（小文本块）
   ↓ 嵌入模型（Embedding Model）
向量列表（float[]）
   ↓ 存储
向量数据库（Milvus/Chroma/FAISS）

【检索生成阶段（在线）】
用户提问（Query）
   ↓ 嵌入模型
查询向量
   ↓ 向量相似度搜索
Top-K 相关 Chunks
   ↓ 构造 Prompt（问题 + 上下文）
   ↓ LLM
最终回答
```

---

### 4.3 文档加载

```python
# 加载 Markdown
from langchain_community.document_loaders import UnstructuredMarkdownLoader
# pip install markdown langchain_community unstructured[md]
docs = UnstructuredMarkdownLoader("file.md", mode="elements").load()

# 加载 Word
from langchain_community.document_loaders import UnstructuredWordDocumentLoader
# pip install unstructured[docx]
docs = UnstructuredWordDocumentLoader("file.docx", mode="elements").load()
```

**Document 对象结构：**
```python
doc.page_content  # 文本内容字符串
doc.metadata      # 元数据字典，如 {"source": "file.pdf", "page": 1}
doc.id            # 可选，文档标识符
```

**加载 PDF 推荐工具：MinerU**

> MinerU 是专业 PDF 解析工具，支持扫描版/双列布局/表格/公式识别，VLM 模型驱动，SOTA 水平
> - GitHub：https://github.com/opendatalab/MinerU
> - 策略：MinerU 解析 → 输出 Markdown → 再用 MarkdownLoader 处理

---

### 4.4 文档切分

**为什么要切分？**
1. 避免无关内容干扰 LLM 生成（**"Lost in the Middle"** 研究发现：相关信息在中间时模型性能显著下降）
2. 避免超出模型最大 Token 限制

**三种切分策略：**

| 策略 | 特点 | 适用 |
|---|---|---|
| 固定字符数/Token数 | 简单快速，可能截断句子 | 对语义要求不高 |
| 递归分隔符切分 | 平衡语义和长度（**推荐**） | 大多数场景 |
| 语义切分 | 语义完整性最好，但慢 | 高质量检索需求 |

**RecursiveCharacterTextSplitter（最常用）：**

```python
# pip install langchain-text-splitters
from langchain_text_splitters import RecursiveCharacterTextSplitter

splitter = RecursiveCharacterTextSplitter(
    separators=["\n\n", "\n", "。", "！", "？", "，", ""],  # 按优先级尝试这些分隔符
    chunk_size=400,    # 每块最大字符数
    chunk_overlap=50,  # 块间重叠字符数（保证边界处语义连贯）
    add_start_index=True,  # 在 metadata 中记录起始位置
)

chunks = splitter.split_documents(docs)
```

**切分过程示意：**
```
原文档
   ↓ 尝试 "\n\n" 分段落
   ↓ 若段落仍 > chunk_size，尝试 "\n" 分句
   ↓ 若句子仍 > chunk_size，尝试 "。" 分子句
   ↓ ... 直到 chunk 足够小
最终 Chunk 列表（每个 chunk 互相重叠 overlap 字符）
```

---

### 4.5 文档嵌入（Embedding）

> 将文本转化为高维向量，使语义相似的文本在向量空间中距离相近

**常用嵌入模型：**

| 模型 | 机构 | 维度 | 最大序列 | 特点 |
|---|---|---|---|---|
| `bge-base-zh-v1.5` | BAAI | 768 | 512 | 中文优化，轻量 |
| `bge-large-zh` | BAAI | 1024 | 512 | 中文高精度 |
| `bge-m3` | BAAI | 1024 | 8192 | 多语言，支持稠密+稀疏双向量 |
| `text-embedding-3-large` | OpenAI | 3072 | 8192 | API调用，高质量 |

```python
# 使用 HuggingFace 本地模型
# pip install sentence-transformers langchain_huggingface
from langchain_huggingface import HuggingFaceEmbeddings

embed_model = HuggingFaceEmbeddings(model_name="./models/bge-base-zh-v1.5")

# 单文本嵌入
query_vector = embed_model.embed_query("你好世界")  # → list[float]

# 批量文本嵌入
vectors = embed_model.embed_documents(["文本1", "文本2"])  # → list[list[float]]
```

---

### 4.6 向量数据库（Milvus）

#### Milvus 架构

```
┌─────────────────────────────────────────────────────┐
│                     Milvus 架构                      │
│                                                     │
│  ┌─────────┐  ┌─────────┐  ┌──────────┐            │
│  │ 查询节点 │  │ 数据节点 │  │ 索引节点  │  独立扩缩容 │
│  └─────────┘  └─────────┘  └──────────┘            │
│       搜索         插入          索引/压缩             │
│                                                     │
│  数据组织：数据库 → Collection（表）→ 实体（行）        │
└─────────────────────────────────────────────────────┘
```

#### 稠密向量 vs 稀疏向量

```
稠密向量（Dense Vector）：
  - 由 Transformer Encoder（如 BGE）生成
  - 表示文本整体语义
  - 格式：[0.23, -0.41, 0.87, ...]  维度固定（如1024）
  - 索引：HNSW（近似最近邻，速度快）

稀疏向量（Sparse Vector）：
  - 表示文本中关键词及权重
  - 格式：{词ID: 权重, ...}  大量为0
  - 类似升级版 TF-IDF，但由深度学习模型生成
  - 索引：SPARSE_INVERTED_INDEX（倒排索引）
```

#### HNSW 索引原理（面试高频）

```
HNSW（分层导航小世界）：

Layer 2（稀疏层）: 少量节点，远距离跳跃
Layer 1（中间层）: 中等节点，中距离跳跃
Layer 0（密集层）: 所有节点，精细搜索

搜索过程：
1. 从顶层入口点开始
2. 贪婪移动到最近邻居
3. 到达局部最小值，下降一层
4. 重复直到底层，得到最终结果

特点：搜索精度高、延迟低，但内存开销大
```

#### 混合检索（Hybrid Search）+ RRF 重排序

```
查询
  ├──→ 稠密向量检索（语义召回）  → Top-K 结果A
  └──→ 稀疏向量检索（关键词召回）→ Top-K 结果B
                    ↓
              RRF 重排序器
         score = Σ 1/(k + rank_i)    k=60（平滑参数）
                    ↓
              最终 Top-K 结果（综合语义+关键词）
```

#### Milvus 完整代码示例

```python
from pymilvus import MilvusClient, DataType

# 1. 连接
client = MilvusClient(uri="http://localhost:19530", token="")

# 2. 建 Schema
schema = (
    MilvusClient.create_schema(auto_id=True)
    .add_field("id", DataType.INT64, is_primary=True)
    .add_field("vector", DataType.FLOAT_VECTOR, dim=1024)   # 稠密向量
    .add_field("sparse_vector", DataType.SPARSE_FLOAT_VECTOR)  # 稀疏向量
    .add_field("text", DataType.VARCHAR, max_length=1500)
    .add_field("metadata", DataType.JSON)
)

# 3. 建索引
index_params = MilvusClient.prepare_index_params()
index_params.add_index("vector", index_type="HNSW", metric_type="L2")
index_params.add_index("sparse_vector", index_type="SPARSE_INVERTED_INDEX", metric_type="IP")

# 4. 创建 Collection
client.create_collection("my_collection", schema=schema, index_params=index_params)

# 5. 插入数据
client.insert("my_collection", data=[
    {"vector": dense_vec, "sparse_vector": sparse_vec, "text": "...", "metadata": {...}}
])

# 6. 混合检索
from pymilvus import AnnSearchRequest, RRFRanker
results = client.hybrid_search(
    collection_name="my_collection",
    reqs=[
        AnnSearchRequest(data=[dense_vec], anns_field="vector", param={"metric_type": "L2"}, limit=5),
        AnnSearchRequest(data=[sparse_vec], anns_field="sparse_vector", param={"metric_type": "IP"}, limit=5),
    ],
    ranker=RRFRanker(k=60),
    limit=5,
    output_fields=["id", "text", "metadata"],
)
```

---

### 4.7 完整 RAG 生成代码

```python
from langchain_openai import ChatOpenAI

def rag_answer(client, query: str) -> str:
    llm = ChatOpenAI(model="gpt-4o-mini")
    
    # 1. 检索
    results = hybrid_search(client, query)  # 返回相关 chunks
    
    # 2. 构造上下文
    context = "\n".join([r["text"] for r in results])
    
    # 3. 生成
    messages = [
        {"role": "system", "content": "你是专业问答机器人。根据上下文回答问题，若上下文无法回答请说明。"},
        {"role": "user", "content": f"上下文：{context}\n\n问题：{query}"}
    ]
    return llm.invoke(messages).content
```

---

## 五、Agents —— 智能体

### 5.1 Agent 核心概念

```
传统 Chain（固定流程）：
输入 → Step1 → Step2 → Step3 → 输出

Agent（动态决策）：
输入 → LLM推理 → 决定是否调用工具？
                      ↓ 是
                   选择工具 → 执行 → 观察结果
                      ↓
                   继续推理 → 是否完成？
                      ↓ 是
                    最终输出
```

**Agent 五大核心组件：**

| 组件 | 作用 |
|---|---|
| **LLM** | 大脑，提供推理/规划/知识 |
| **Memory** | 短期（对话历史）+ 长期（向量库）记忆 |
| **Tools** | 调用外部 API/数据库/搜索引擎 |
| **Planning** | 任务分解、反思与自省 |
| **Action** | 实际执行决策 |

---

### 5.2 工具调用原理（Function Calling）

```
1. 用 JSON Schema 定义工具（名称、描述、参数）
   ↓
2. 连同用户问题一起发给 LLM
   ↓
3. LLM 决定是否调用工具，返回 tool_call（工具名+参数）
   ↓
4. 代码层执行实际工具函数，得到结果
   ↓
5. 将结果作为 ToolMessage 加入历史，再次调用 LLM
   ↓
6. LLM 基于工具结果生成最终回答
```

#### 用 LangChain 定义工具

```python
from langchain.tools import tool
from pydantic import BaseModel, Field

# 可选：定义参数的详细描述
class GetWeatherArgs(BaseModel):
    city: str = Field(description="城市名称")
    date: str = Field(description="日期，格式 YYYY-MM-DD")

@tool(args_schema=GetWeatherArgs)
def get_weather(city: str, date: str) -> str:
    """获取指定城市在指定日期的天气"""  # 函数文档字符串 = 工具描述
    return f"{city} 在 {date} 天气多云，有下雨的可能性"
```

> ⚠️ **关键**：工具的函数文档字符串（`docstring`）会作为工具描述传给 LLM，**必须写清楚**！

---

### 5.3 构建 Agent

```python
# pip install langchain langchain-tavily
from langchain.agents import create_agent
from langchain_tavily import TavilySearch
from langchain.chat_models import init_chat_model

llm = init_chat_model(model="gpt-4o-mini", model_provider="openai")
tools = [TavilySearch(max_results=5)]

agent = create_agent(
    model=llm,
    tools=tools,
    system_prompt="你是一个助手，需要调用工具帮助用户。",
)

# 调用（返回完整结果）
result = agent.invoke({"messages": [{"role": "user", "content": "今天北京天气？"}]})
print(result['messages'][-1].content)

# 流式调用（推荐，可看到中间步骤）
for chunk in agent.stream({"messages": [{"role": "user", "content": "今天北京天气？"}]}):
    print(chunk, end="\n\n")
```

**`create_agent` 关键参数：**

| 参数 | 说明 |
|---|---|
| `model` | LLM 实例 |
| `tools` | 工具列表 |
| `system_prompt` | 系统提示词 |
| `checkpointer` | 短期记忆（保存对话历史） |
| `middleware` | 中间件（总结/人工审核） |
| `response_format` | 结构化输出 Schema |

---

### 5.4 Agent 添加记忆（短期记忆）

```
Thread 机制：
┌──────────────────────────────────────────────────┐
│  thread_id: "user_001"                           │
│  消息列表：[msg1, msg2, msg3, ...]               │
├──────────────────────────────────────────────────┤
│  thread_id: "user_002"                           │
│  消息列表：[msg1, msg2, ...]                     │
└──────────────────────────────────────────────────┘
不同 thread_id 完全隔离，互不影响
```

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

agent = create_agent(
    model=llm,
    tools=tools,
    checkpointer=checkpointer,  # ← 开启短期记忆
)

# 第一次对话
agent.invoke(
    {"messages": [{"role": "user", "content": "今天北京天气？"}]},
    config={"configurable": {"thread_id": "abc123"}}  # ← 指定 thread
)

# 第二次对话（同 thread_id → 记得上一次对话）
agent.invoke(
    {"messages": [{"role": "user", "content": "我刚才问了什么？"}]},
    config={"configurable": {"thread_id": "abc123"}}  # ← 同一个 thread
)
```

---

### 5.5 中间件（Middleware）

#### 中间件执行流程

```
用户输入
   ↓ before_agent
[Agent Start]
   ↓ before_model
[LLM 推理]
   ↓ after_model
判断是否调用工具？
   ↓ wrap_tool_call
[工具执行]
   ↓ after_tool
继续 LLM 推理...
   ↓ after_agent
最终输出
```

#### SummarizationMiddleware（对话历史压缩）

```python
from langchain.agents.middleware import SummarizationMiddleware

summary_middleware = SummarizationMiddleware(
    model=ChatOpenAI(model="gpt-4o-mini"),
    trigger=("messages", 10),  # 消息超过10条时自动压缩
    # trigger=("fraction", 0.5)  # Token 达到最大输入的50%时压缩
    # trigger=("tokens", 3000)   # Token 绝对数超过3000时压缩
)
```

#### HumanInTheLoopMiddleware（人工审核）

> 在调用敏感工具前暂停，等待人工审批

```python
from langchain.agents.middleware.human_in_the_loop import HumanInTheLoopMiddleware

hitl_middleware = HumanInTheLoopMiddleware(
    interrupt_on={
        "transfer_money": True,  # 需要人工审核
        "get_weather": False,    # 自动通过
    }
)

agent = create_agent(
    model=llm,
    tools=[get_weather, transfer_money],
    middleware=[hitl_middleware],
    checkpointer=InMemorySaver(),  # 中断需要保存状态，必须配 checkpointer
)
```

**中断恢复流程：**
```python
# 检测到中断（__interrupt__ 键存在于 result 中）
if "__interrupt__" in result:
    # 人工审核...
    # 恢复执行
    from langgraph.types import Command
    result = await agent.ainvoke(
        Command(resume={"decisions": [{"type": "approve"}]}),
        config=config
    )
```

---

## 六、MCP —— 模型上下文协议

### 6.1 什么是 MCP

> **MCP（Model Context Protocol）**：标准化 LLM 与外部工具通信的开源协议，类似 USB-C 标准

**一次集成，处处可用**：工具提供方只需实现 MCP 服务端，任何兼容 MCP 的 AI 应用都能使用

---

### 6.2 MCP 架构与工作流程

```
┌─────────────────────────────────────────────────────────┐
│                      MCP 架构                            │
│                                                         │
│  ┌──────────────────────────────────────┐               │
│  │            MCP Host (AI 应用)         │               │
│  │  ┌────────────┐  ┌────────────┐      │               │
│  │  │ MCP Client │  │ MCP Client │ ...  │               │
│  │  └─────┬──────┘  └─────┬──────┘      │               │
│  └────────┼───────────────┼─────────────┘               │
│           │ JSON-RPC       │ JSON-RPC                    │
│    ┌──────▼──────┐  ┌──────▼──────┐                     │
│    │ MCP Server  │  │ MCP Server  │                     │
│    │ (本地工具)   │  │ (远程服务)   │                     │
│    └─────────────┘  └─────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

**工作流程：**
```
1. Host 启动 → 连接 MCP Server → 获取工具列表（握手）
2. 用户提问 → Host 将问题 + 工具描述发给 LLM
3. LLM 决策 → 是否调用工具，返回调用指令
4. Host 通过 MCP 协议 → MCP Server 执行工具
5. Server 返回结果 → Host → LLM 继续推理 → 最终回答
```

---

### 6.3 传输协议对比

| 协议 | 适用场景 | 特点 |
|---|---|---|
| **Stdio** | 本地工具（子进程通信） | 简单，适合本地开发 |
| **Streamable HTTP** | 远程服务 | 支持流式、认证，主流选择 |
| SSE | 旧版 | 逐渐被 Streamable HTTP 取代 |

---

### 6.4 MCP 服务端开发

```python
# pip install mcp
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("MyTools")

@mcp.tool()
def add(a: int, b: int) -> int:
    """计算两数之和"""
    return a + b

@mcp.resource("greeting://default")
def get_greeting() -> str:
    return "Hello from MCP!"

@mcp.prompt()
def greet_user(name: str) -> str:
    return f"为 {name} 写一句友善的问候"

if __name__ == "__main__":
    mcp.run(transport="stdio")          # 本地模式
    # mcp.run(transport="streamable-http")  # HTTP 模式，默认 127.0.0.1:8000
```

---

### 6.5 LangChain Agent 接入 MCP

```python
# pip install langchain_mcp_adapters
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI

# 配置 MCP 服务端（此处以魔搭 12306 MCP 为例）
client = MultiServerMCPClient({
    "12306-mcp": {
        "transport": "streamable_http",
        "url": "https://mcp.api-inference.modelscope.net/xxx/mcp"
    }
})

async def main():
    tools = await client.get_tools()  # 获取 MCP 工具（已封装为 LangChain 格式）
    
    agent = create_agent(
        model=ChatOpenAI(model="gpt-4o-mini"),
        tools=tools,
    )
    
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "查询明天北京到上海的高铁"}]},
        config={"configurable": {"thread_id": "1"}}
    )
    print(result['messages'][-1].content)

asyncio.run(main())
```

> ⚠️ `MultiServerMCPClient` 只支持异步调用，Agent 也需要用 `ainvoke`

---

### 6.6 MCP vs 本地工具对比

| | 本地工具（`@tool`） | MCP 工具 |
|---|---|---|
| 实现方式 | Python 函数 + 装饰器 | 独立 MCP Server |
| 适用场景 | 私有业务逻辑 | 标准化外部服务 |
| 跨应用复用 | 不易 | 一次实现，处处使用 |
| 示例 | 数据库查询、业务计算 | 搜索引擎、12306、地图 |

---

## 七、面试重点速查

### 🎯 必背知识点

**1. LangChain 有哪些核心模块？**
> Model I/O（提示模板+模型调用+输出解析）、Chains（LCEL链式组合）、Retrieval（RAG）、Agents（工具调用+自主决策）

**2. 什么是 Runnable 接口？为什么重要？**
> LangChain 核心抽象，所有组件（LLM、Prompt、Parser）均实现此接口，统一暴露 invoke/batch/stream 方法。使得 LCEL 的 `|` 管道语法成为可能，组件可任意组合。

**3. RAG 的完整流程？**
> 索引阶段：加载→切分→嵌入→存入向量库；检索阶段：问题嵌入→向量相似搜索→Top-K chunks→构造 Prompt→LLM 生成

**4. 稠密向量和稀疏向量的区别？**
> 稠密：Transformer 生成，表达整体语义，适合语义检索；稀疏：深度学习权重化关键词，表达词级特征，适合关键词匹配。混合检索（Hybrid Search）结合两者效果最好。

**5. HNSW 索引的原理？**
> 多层图结构，顶层稀疏用于快速跳跃，底层密集用于精细搜索。搜索时从顶层入口贪婪下降，最终在底层找到最近邻。特点：高精度、低延迟，但内存大。

**6. Agent 和 Chain 的区别？**
> Chain：固定顺序执行，适合确定性流程；Agent：LLM 动态决策，自主选择工具和步骤，适合开放性多步骤任务

**7. Function Calling 工作原理？**
> 将工具定义（JSON Schema）和用户问题一起传给 LLM → LLM 返回工具调用指令 → 代码执行工具 → 结果作为 ToolMessage 再次传给 LLM → 最终回答

**8. MCP 是什么？解决什么问题？**
> 标准化 LLM 和外部工具通信的协议，类似 USB-C。解决：每个 AI 应用需要单独对接每个工具的问题，实现"一次实现，处处可用"

**9. `with_structured_output` 的作用？**
> LLM 方法，传入 Pydantic 模型作为 Schema，返回一个新的 LLM 对象，调用后直接返回 Pydantic 实例，无论底层厂商差异，接口统一

**10. Thread ID 的作用？**
> 在有 checkpointer 的 Agent 中，thread_id 标识一个独立的对话会话，同一 thread_id 的多次调用共享消息历史，不同 thread_id 完全隔离

---

### 📦 常用包安装速查

```bash
pip install langchain                    # 主包
pip install langchain-core               # 核心抽象
pip install langchain-openai             # OpenAI 集成
pip install langchain-ollama             # Ollama 本地模型
pip install langchain-text-splitters     # 文本切分
pip install langchain-community          # 社区集成（文档加载器等）
pip install langchain-huggingface        # HuggingFace 模型
pip install langchain-mcp-adapters       # MCP 适配器
pip install langchain-tavily             # Tavily 搜索工具
pip install langgraph                    # Agent 运行时（底层）
pip install pymilvus                     # Milvus 向量数据库
pip install sentence-transformers        # 嵌入模型
pip install python-dotenv               # 环境变量管理
```

---

### 🗺️ 技术选型速查

| 需求 | 推荐方案 |
|---|---|
| 在线 LLM 调用 | `ChatOpenAI` / `init_chat_model` |
| 本地 LLM | Ollama + `ChatOllama`（原型），vLLM（生产） |
| 文本嵌入（中文） | `bge-base-zh-v1.5` 或 `bge-m3` |
| 向量数据库（生产） | Milvus Standalone |
| 向量数据库（原型） | Chroma 或 FAISS |
| PDF 解析 | MinerU |
| 文档切分 | `RecursiveCharacterTextSplitter` |
| 外部搜索工具 | Tavily Search |
| 外部服务接入 | MCP 协议 |
| Agent 调试 | LangSmith（环境变量配置即可，无需改代码） |

---

