

> 📚 来源：尚硅谷大模型技术之LangGraph V1.1.0  
> 🎯 目标：深度掌握 LangGraph，构建生产级 Agent 应用  
> 🔗 关联笔记：[[大模型应用/LangChain|LangChain]]

---

## 目录

- [[#一、LangGraph 概述]]
- [[#二、State —— 状态（核心概念）]]
- [[#三、Node —— 节点]]
- [[#四、Edge —— 边]]
- [[#五、Checkpointer —— 状态持久化]]
- [[#六、Pregel —— 底层运行引擎]]
- [[#七、流式输出（Stream）]]
- [[#八、interrupt —— 人工介入]]
- [[#九、面试重点速查]]

---

## 一、LangGraph 概述

### 1.1 什么是 LangGraph

> LangGraph 是一个**低级编排框架和运行时环境**，用于构建、管理和部署长期运行的**有状态智能体（Agents）**。

**核心理念：将 Agent 工作流建模为图（Graph）**

```
三大基本元素：

  节点（Node）          边（Edge）            状态（State）
  ──────────           ──────────           ──────────
  计算单元              转换逻辑              共享数据
  LLM调用              决定执行流程           贯穿整个图
  工具执行              普通边/条件边          节点读写的对象
  自定义逻辑
```

**核心能力：**
- **持久化执行**：从故障中恢复，长时间运行
- **人机协作**：任意时刻检查/修改 Agent 状态（interrupt）
- **记忆管理**：短期（Checkpointer）+ 长期记忆
- **流式处理**：专为流式工作流设计
- **生产级部署**：可扩展的有状态基础设施

**相关链接：**
- GitHub：https://github.com/langchain-ai/langgraph
- 官方文档：https://docs.langchain.com/oss/python/langgraph/overview

---

### 1.2 LangGraph vs LangChain（重要！）

> 一句话总结：当 LLM 应用需要「**有状态、可循环、可分支的多步骤控制流**」时，LangChain 很难优雅完成，而必须用 **LangGraph**。

| 特性 | LangGraph | LangChain |
|---|---|---|
| 抽象级别 | 低级，细粒度控制 | 高级，开箱即用 |
| 状态管理 | **内置状态机和检查点** | 需自行管理 |
| 执行模型 | **基于图，支持并行/循环** | 线性链式执行 |
| 持久化 | **原生支持** | 需额外实现 |
| 适用场景 | 复杂、有状态的 Agent 应用 | 简单的链式调用 |

**LangGraph 主要应用场景：**
- 复杂的多智能体系统
- 需要长期记忆的应用
- 需要人工审核的工作流（Human-in-the-Loop）
- 后台处理任务 & 实时交互
- 需要精细控制的定制化 Agent 编排

---

### 1.3 快速入门：五步构建图

```
第一步：定义 State（状态数据结构）
   ↓
第二步：定义节点函数（Node Functions）
   ↓
第三步：创建 StateGraph，添加节点和边
   ↓
第四步：编译图（compile）→ 得到可调用对象
   ↓
第五步：调用图（invoke / stream）
```

**完整示例（RAG 问答系统）：**

```python
from typing import TypedDict
from langgraph.constants import START, END
from langgraph.graph import StateGraph

# 第一步：定义状态
class MyState(TypedDict):
    query: str
    rag_result: str
    web_search_result: str
    final_answer: str

# 第二步：定义节点函数（只返回增量更新！）
def rag_search_node(state: MyState):
    return {"rag_result": f"关于{state['query']}的RAG结果"}

def web_search_node(state: MyState):
    return {"web_search_result": f"关于{state['query']}的网络结果"}

def final_answer_node(state: MyState):
    return {"final_answer": f"LLM综合了{state['rag_result']}和{state['web_search_result']}"}

# 第三步：构建图
graph = StateGraph(state_schema=MyState)
graph.add_node(rag_search_node)
graph.add_node(web_search_node)
graph.add_node(final_answer_node)

# 并行执行：START 同时连接两个节点
graph.add_edge(START, "rag_search_node")
graph.add_edge(START, "web_search_node")
graph.add_edge("rag_search_node", "final_answer_node")
graph.add_edge("web_search_node", "final_answer_node")
graph.add_edge("final_answer_node", END)

# 第四步：编译
compiled_graph = graph.compile()

# 可视化图结构
# pip install grandalf
graph_structure = compiled_graph.get_graph()
print(graph_structure.draw_ascii())

# 第五步：调用
result = compiled_graph.invoke({"query": "如何使用LangGraph"})
print(result)
```

**对应图结构：**
```
         START
        /     \
  rag_search  web_search   ← 并行执行
        \     /
      final_answer
           |
          END
```

---

## 二、State —— 状态（核心概念）

> 💡 State 是 LangGraph 最重要的概念，代表图的**数据目标**和所有节点**共享的全局数据**。

### 2.1 状态的三种定义方式

```python
# 方式一：TypedDict（推荐）
from typing import TypedDict

class MyState(TypedDict):
    query: str
    result: str
    messages: list

# 方式二：Pydantic BaseModel（需要运行时数据校验时用）
from pydantic import BaseModel

class MyState(BaseModel):
    query: str
    result: str

# 方式三：dataclass
from dataclasses import dataclass

@dataclass
class MyState:
    query: str
    result: str
```

> ✅ **推荐使用 TypedDict**，简洁且与 LangGraph 配合最好

---

### 2.2 输入输出数据隔离

> 可以精细控制图的输入字段和输出字段，实现接口隔离

```
全局状态（state_schema）：所有节点可读写的完整状态空间
       ┌─────────────────────────────────────────┐
       │  query / rag_result / web_result /      │
       │  final_answer / internal_debug_info ... │
       └─────────────────────────────────────────┘
              ↑                    ↓
    input_schema（子集）    output_schema（子集）
    仅暴露：query           仅返回：final_answer
```

```python
class MyStateFull(TypedDict):
    query: str
    rag_result: str
    web_search_result: str
    final_answer: str
    internal_key: str  # 内部使用，不暴露给调用方

class InputSchema(TypedDict):
    query: str          # 调用方只需传 query

class OutputSchema(TypedDict):
    final_answer: str   # 调用方只拿 final_answer

# 初始化时指定三个 schema
graph = StateGraph(
    state_schema=MyStateFull,
    input_schema=InputSchema,
    output_schema=OutputSchema
)
```

---

### 2.3 节点间数据隔离（私有状态）

> 某个节点可以接收与全局 State 不同的私有状态，用于节点内部逻辑

```python
class MyState(TypedDict):        # 全局状态
    query: str
    final_answer: str

class SearchState(TypedDict):    # 某个节点的私有状态
    rag_result: str
    web_search_result: str

# 节点函数声明使用私有状态
def final_answer_node(state: SearchState):  # 接收 SearchState，而非 MyState
    return {"final_answer": f"基于{state['rag_result']}的答案"}
```

---

### 2.4 Reducer 函数（状态合并机制）

> 💡 **Reducer 是 LangGraph 状态更新的核心机制**，定义了"节点输出的增量状态如何与全局状态合并"

```
节点执行后返回增量状态
          ↓
    Reducer 函数
    (left, right) → merged
          ↓
    全局状态更新
```

**三种 Reducer 行为：**

```python
from typing import Annotated, List
from typing_extensions import TypedDict
from langchain_core.messages import BaseMessage

# 1. 默认行为：覆盖（未指定 Reducer）
class DefaultState(TypedDict):
    foo: int        # 节点返回新值 → 直接覆盖旧值
    bar: List[str]  # 节点返回新列表 → 直接替换旧列表

# 2. 内置 Reducer：add_messages（消息追加）
from langgraph.graph.message import add_messages

class AgentState(TypedDict):
    # Annotated[类型, reducer函数] 语法
    messages: Annotated[List[BaseMessage], add_messages]
    # 节点返回新消息 → 追加到列表，而非覆盖

# 3. 自定义 Reducer
import operator

class AggState(TypedDict):
    # operator.add 对列表是追加操作
    aggregate: Annotated[list, operator.add]

# 或者完全自定义
def my_merge(left: list, right: list) -> list:
    return left + right  # 自定义合并逻辑

class CustomState(TypedDict):
    items: Annotated[list, my_merge]
```

**Reducer 执行时机：**
```
graph.invoke({"messages": [HumanMessage("你好")]})
                                ↓
             初始状态：messages = [HumanMessage("你好")]
                                ↓
             node_1 返回：{"messages": [AIMessage("你好！")]}
                                ↓
             add_messages Reducer：
             旧值 [HumanMessage] + 新值 [AIMessage] → [HumanMessage, AIMessage]
                                ↓
             更新后状态：messages = [HumanMessage("你好"), AIMessage("你好！")]
```

> ⚠️ **关键规则**：节点只能返回**增量更新**（只包含当前节点修改的键），不能返回整个 state 对象！否则会导致其他键被错误覆盖。

---

## 三、Node —— 节点

### 3.1 节点函数签名

节点是 Python 函数，LangGraph 运行时会自动注入以下参数：

```python
from langchain_core.runnables import RunnableConfig
from langgraph.runtime import Runtime

def my_node(
    state: MyState,          # 必须：当前全局状态
    config: RunnableConfig,  # 可选：配置信息（thread_id、user_id 等）
    runtime: Runtime,        # 可选：运行时对象（stream_writer、context 等）
) -> dict:
    # 1. 从 config 中读取配置参数
    user_id = config.get("configurable", {}).get("user_id", "guest")

    # 2. 从 runtime.context 中获取注入的依赖
    llm = runtime.context["llm"]
    db = runtime.context["db"]

    # 3. 使用 runtime.stream_writer 向图外推送自定义数据
    writer = runtime.stream_writer
    writer({"status": "thinking", "message": "正在处理..."})

    # 4. 返回增量状态
    return {"result": llm.invoke(state["query"])}
```

**依赖注入示例（runtime.context）：**
```python
# 调用时传入 context
result = graph.invoke(
    initial_state,
    config={"configurable": {"user_id": "vip_001"}},
    context={"llm": my_llm, "db": my_db}  # ← 注入外部依赖
)
```

---

### 3.2 特殊节点 START / END

```python
from langgraph.constants import START, END
# START = "__start__"  入口节点，将用户输入发送到图中
# END   = "__end__"   终止节点，表示执行完成

# 从 START 连出表示"第一批执行的节点"
graph.add_edge(START, "node_a")

# 连到 END 表示"该路径的执行结束"
graph.add_edge("node_z", END)
```

---

### 3.3 节点缓存

> 对于**输入相同、结果确定**的耗时节点（LLM 调用、数据库查询），可配置缓存避免重复计算

```python
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy

def expensive_node(state: State) -> dict:
    # 耗时操作...
    return {"result": state["x"] * 2}

# 为节点配置缓存策略
builder.add_node(
    "expensive_node",
    expensive_node,
    cache_policy=CachePolicy(ttl=10)  # 缓存10秒
)

# 编译时传入缓存后端
graph = builder.compile(
    cache=InMemoryCache(),   # 内存缓存（也支持 Redis、SQLite）
    checkpointer=checkpointer
)

# 第一次调用：实际执行节点（慢）
graph.invoke({"x": 5}, config={"configurable": {"thread_id": "1"}})

# 第二次调用相同输入：直接从缓存读取（快）
graph.invoke({"x": 5}, config={"configurable": {"thread_id": "2"}})
```

**缓存 Key 生成：** 默认使用 pickle 对输入做 hash，可通过 `CachePolicy(key_func=...)` 自定义

---

### 3.4 节点重试策略

> 适用于不稳定的节点：LLM API 调用、外部数据库查询等

```python
from langgraph.types import RetryPolicy

# 默认重试策略：对大多数异常重试（ValueError/TypeError 等不会重试）
builder.add_node(
    "unstable_api",
    unstable_api_call,
    retry_policy=RetryPolicy(max_attempts=5)  # 最多重试5次
)

# 自定义：只对特定异常重试
builder.add_node(
    "custom_retry",
    some_node,
    retry_policy=RetryPolicy(
        max_attempts=3,
        retry_on=(ConnectionError, TimeoutError),  # 只对这些异常重试
    )
)
```

**默认不重试的异常类型（重要！）：**
`ValueError`, `TypeError`, `ArithmeticError`, `ImportError`, `LookupError`, `NameError`, `SyntaxError`, `RuntimeError`, `OSError` 等逻辑错误

---

## 四、Edge —— 边

### 4.1 边的本质

> 边本质上定义了 Pregel 中**节点订阅的通道**和**节点执行后更新的通道**，从而决定执行顺序。

**两种边类型：**

| 类型  | API                                               | 说明              |
| --- | ------------------------------------------------- | --------------- |
| 普通边 | `add_edge(from, to)`                              | 固定连接，A 执行完必定到 B |
| 条件边 | `add_conditional_edges(from, router_fn, mapping)` | 动态路由，由函数决定下一个节点 |

---

### 4.2 普通边

```python
# 串行
graph.add_edge("node_a", "node_b")

# 并行（同时从 START 出发）
graph.add_edge(START, "node_a")
graph.add_edge(START, "node_b")  # a 和 b 并行执行

# 多路汇聚（b 和 c 都完成后，d 才执行）
graph.add_edge("node_b", "node_d")
graph.add_edge("node_c", "node_d")
```

---

### 4.3 条件边（重点！）

```
node_a 执行完
      ↓
 router_fn(state)
      ↓
 返回字符串（路由目标）
      ↓
 按 mapping 映射到实际节点名
```

```python
from typing import Literal

# 路由函数：根据状态返回路由目标字符串
def route_condition(state: GraphState) -> Literal["node_b", "node_c"]:
    if state["value"] % 2 == 0:
        return "go_to_b"   # 偶数走 B
    else:
        return "go_to_c"   # 奇数走 C

# 添加条件边
graph.add_conditional_edges(
    "node_a",          # 源节点
    route_condition,   # 路由函数
    {                  # 返回值 → 实际节点名 的映射（可省略，直接用节点名）
        "go_to_b": "node_b",
        "go_to_c": "node_c",
    }
)
```

---

### 4.4 可控循环（ReAct 模式）

> Agent 的经典模式：LLM 决策 → 工具调用 → 观察结果 → 再次决策，形成循环

```
START
  ↓
LLM 节点
  ↓ add_conditional_edges
  ├──→ "call_tool" → 工具节点 ──→ LLM 节点（循环）
  └──→ END（决策完成）
```

```python
def should_continue(state) -> Literal["tools", "__end__"]:
    messages = state["messages"]
    last_msg = messages[-1]
    # 如果 LLM 决定调用工具，继续循环
    if last_msg.tool_calls:
        return "tools"
    # 否则结束
    return "__end__"

graph.add_conditional_edges("llm_node", should_continue)
graph.add_edge("tools_node", "llm_node")  # 工具执行完回到 LLM
```

**⚠️ 递归限制（防止死循环）：**
```python
# 默认25步，超过抛出 GraphRecursionError
result = graph.invoke(
    initial_state,
    config={"recursion_limit": 50}  # 自定义递归限制
)

from langgraph.errors import GraphRecursionError
try:
    result = graph.invoke(initial_state, config={"recursion_limit": 6})
except GraphRecursionError as e:
    print(f"超出递归限制：{e}")
```

---

## 五、Checkpointer —— 状态持久化

### 5.1 为什么需要 Checkpointer？

```
问题一：多次调用间保持上下文（短期记忆）
  第1次 invoke → 初始化空状态 → 写入消息历史
  第2次 invoke → 又初始化空状态 → 丢失第1次历史！❌

问题二：节点故障后断点续传
  node_1 ✅ → node_2 ❌ 报错 → 整个流程失败
  修复 node_2 后，需要从 node_1 之后继续，不想重新执行 node_1 ❌

解决方案：Checkpointer
  每个超步执行后，将状态保存到外部存储
  下次调用时，从指定 thread_id 恢复状态，而不是初始化空状态
```

---

### 5.2 Checkpointer 工作原理

```
graph.invoke(input, config={"configurable": {"thread_id": "T1"}})
                                ↓
            LangGraph 检查：thread_id="T1" 是否有历史 checkpoint？
                    ↓ 有                         ↓ 没有
            加载最新 checkpoint             使用传入的 input 初始化
                    ↓
            合并 input（如果有）
                    ↓
            执行图，每个超步后写入 checkpoint
                    ↓
            结束，最终状态写入 checkpoint
```

**Thread ID 机制：**
```
thread_id="user_A_session_1" → [msg1, msg2, msg3, ...]  ← 对话A
thread_id="user_A_session_2" → [msg1, ...]              ← 对话A的新会话
thread_id="user_B_session_1" → [msg1, msg2, ...]        ← 对话B
完全隔离，互不影响
```

---

### 5.3 Checkpointer 类型

| 类型 | 使用场景 | 代码 |
|---|---|---|
| `InMemorySaver` | 开发/测试，进程内存 | `from langgraph.checkpoint.memory import InMemorySaver` |
| `SqliteSaver` | 本地持久化，轻量生产 | `from langgraph.checkpoint.sqlite import SqliteSaver` |
| PostgresSaver | 生产环境，分布式 | `from langgraph.checkpoint.postgres import PostgresSaver` |

---

### 5.4 使用 Checkpointer 实现短期记忆

```python
from langgraph.checkpoint.memory import InMemorySaver
from langchain.agents import create_agent
from langchain_openai import ChatOpenAI

checkpointer = InMemorySaver()

agent = create_agent(
    model=ChatOpenAI(model="gpt-4o-mini"),
    tools=[...],
    checkpointer=checkpointer,  # ← 启用短期记忆
)

# 同一 thread_id → 保持上下文
agent.invoke(
    {"messages": "北京2026-01-14天气怎么样"},
    config={"configurable": {"thread_id": "user_1_session_1"}}
)

# 第二次调用，LLM 能记住上一次对话
agent.invoke(
    {"messages": "适合出去玩吗"},
    config={"configurable": {"thread_id": "user_1_session_1"}}  # ← 同一 thread
)

# 新对话：换一个 thread_id
agent.invoke(
    {"messages": "你好"},
    config={"configurable": {"thread_id": "user_1_session_2"}}  # ← 新会话
)
```

---

### 5.5 故障恢复（断点续传）

```python
import sqlite3
from langgraph.checkpoint.sqlite import SqliteSaver

# 1. 使用 SQLite 持久化（进程重启后数据不丢失）
conn = sqlite3.connect("./checkpoints.db", check_same_thread=False)
memory = SqliteSaver(conn)

graph = builder.compile(checkpointer=memory)

config = {"configurable": {"thread_id": "job_123"}}

# 第一次运行：node_2 报错
try:
    graph.invoke({}, config=config)
except Exception as e:
    print(f"node_2 出错：{e}")
    # 此时 node_1 的状态已保存到 SQLite

# 修复 node_2 后，断点续传
# 关键：传 None 作为输入 + 相同 thread_id
graph.invoke(None, config=config)  # ← 从 checkpoint 恢复，跳过已成功的节点
```

---

### 5.6 获取历史状态

```python
# 获取最近一个时间步的状态
last_state = graph.get_state(config={"configurable": {"thread_id": "1"}})

# 获取所有历史状态（倒序，最新在前）
all_states = graph.get_state_history(config={"configurable": {"thread_id": "1"}})

for snapshot in all_states:
    print(snapshot.values)        # 当前状态数据
    print(snapshot.next)          # 接下来要执行的节点名
    print(snapshot.config)        # 获取此快照的配置
    print(snapshot.metadata)      # 元数据（step、source等）
    print(snapshot.parent_config) # 父快照的配置
    print(snapshot.interrupts)    # 待解决的中断
```

**StateSnapshot 结构：**
```
StateSnapshot:
  ├── values: dict        ← 当前状态值
  ├── next: tuple         ← 下一个要执行的节点（断点续传关键！）
  ├── config: dict        ← 当前快照的配置（含 checkpoint_id）
  ├── metadata: dict      ← 元数据（step、source 等）
  ├── parent_config: dict ← 父快照配置
  └── interrupts: tuple   ← 待处理的中断
```

---

## 六、Pregel —— 底层运行引擎

### 6.1 Pregel 核心概念

> LangGraph 底层运行算法，控制整个图从开始到结束的迭代执行过程

**两大核心组件：**

```
┌──────────────────────────────────────────────────────────┐
│                    Pregel 运行引擎                         │
│                                                          │
│  Actors（节点）               Channels（通道）             │
│  ──────────────               ────────────               │
│  PregelNode 实例              节点间通信的媒介              │
│  订阅 channels（读）           每个通道有：                 │
│  写入 channels（写）           - 值类型                    │
│  实现 Runnable 接口            - 更新类型                  │
│                               - 更新函数（Reducer）        │
└──────────────────────────────────────────────────────────┘
```

---

### 6.2 超步（SuperStep）执行流程

> 与传统 DAG 不同，LangGraph 基于"超步"迭代执行，不是"所有上游节点完成才执行下游"

```
超步 = Plan → Execute → Update 的一次循环

Plan 阶段：
  确定本步要执行哪些 Actors
  第一步：选择订阅 __start__ 通道的 Actors
  后续步：选择订阅上一步被更新通道的 Actors

Execute 阶段：
  并行执行所有选定的 Actors
  直到所有完成 / 其中一个失败 / 超时

Update 阶段：
  用本步 Actors 输出更新对应 Channels

循环，直到没有 Actors 被选中，或达到最大步数
```

**以 a→b,c; b→b_2→d; c→d 的图为例：**

```
Step -1: START 执行，写入 branch:to:a
Step  0: 节点 a 执行 → 写入 branch:to:b, branch:to:c
Step  1: 节点 b、c 并行执行 → 写入 branch:to:b_2, branch:to:d
Step  2: 节点 b_2、d 执行 → b_2 再写入 branch:to:d
Step  3: 节点 d 再次执行（收到 b_2 的更新）
Step  4: 无 Actor 被激活，图结束
```

**关键推论：** 节点 `d` 被执行了**两次**（Step2 和 Step3），因为它订阅的通道被更新了两次！这与传统 DAG 行为不同。

---

### 6.3 Checkpoint 保存时机

> 每个超步结束后（包括 START 节点执行后），LangGraph 自动保存一次 checkpoint

```
Start checkpoint (step=-1)：START 节点执行后
Step0 checkpoint：第一批节点执行后
Step1 checkpoint：第二批节点执行后
...
Final checkpoint：END 节点执行后
```

---

## 七、流式输出（Stream）

### 7.1 为什么需要流式输出？

```
Agent 执行过程可能很长：
  节点1 → 节点2 → LLM 生成（耗时10秒）→ 节点3 → 结束

如果用 invoke：用户等待整个过程结束才看到结果
如果用 stream：用户实时看到每一步的进展和 LLM 的 token
```

---

### 7.2 六种 Stream 模式

| 模式 | 返回内容 | 典型用途 |
|---|---|---|
| `values` | 每步执行后的**完整状态** | 监控整体状态变化 |
| `updates` | 每步执行后的**增量更新** | 监控节点输出 |
| `custom` | 节点内部通过 `writer()` 发送的**自定义数据** | 推送进度提示 |
| `messages` | LLM 生成的 **Token 流**（二元组） | 打字机效果 |
| `debug` | 所有调试信息 | 开发调试 |
| 混合模式 | 多种模式同时输出（三元组） | 同时监控多种数据 |

---

### 7.3 各模式代码示例

```python
initial_state = {"messages": [], "current_step": "start"}

# ── 模式1：values（完整状态）──
for state in graph.stream(initial_state, stream_mode="values"):
    print(f"当前步骤: {state.get('current_step')}")
    print(f"状态键: {list(state.keys())}")

# ── 模式2：updates（增量更新）──
for update in graph.stream(initial_state, stream_mode="updates"):
    # update = {"节点名": {"修改的key": "新值", ...}}
    node_name = list(update.keys())[0]
    print(f"节点 [{node_name}] 更新: {update[node_name]}")

# ── 模式3：custom（自定义数据）──
# 节点内部需要用 runtime.stream_writer 发送
for data in graph.stream(initial_state, stream_mode="custom"):
    print(f"自定义数据: {data}")

# ── 模式4：messages（LLM Token）──
for chunk, metadata in graph.stream(initial_state, stream_mode="messages"):
    node = metadata.get("langgraph_node")
    print(f"[{node}] token: {chunk.content}", end="")

# ── 模式5：debug ──
for event in graph.stream(initial_state, stream_mode="debug"):
    print(f"调试事件: {event['type']}")

# ── 模式6：混合模式 ──
for mode, data in graph.stream(initial_state, stream_mode=["updates", "custom"]):
    if mode == "updates":
        print(f"[更新] {list(data.keys())[0]}")
    elif mode == "custom":
        print(f"[自定义] {data}")
```

**在节点内部发送自定义数据：**
```python
def my_node(state: State, runtime: Runtime):
    writer = runtime.stream_writer
    
    for step in ["分析中...", "检索中...", "生成中..."]:
        writer({"progress": step})  # 实时推送进度
    
    return {"result": "最终结果"}
```

---

## 八、interrupt —— 人工介入

### 8.1 interrupt 原理

```
                    ┌─────────────────────────────────────┐
                    │           图执行                     │
                    │                                     │
  第一次 invoke ──→  │  review_node                        │
                    │    ...                              │
                    │    user_input = interrupt(data) ←── │── 暂停！
                    │    ...（暂停，不继续）                │
                    └─────────────────────────────────────┘
                              ↓
              result["__interrupt__"][0].value  ← 包含要审核的数据
                              ↓
                    用户审核/决策
                              ↓
  第二次 invoke ──→  Command(resume=user_decision)
                    │    user_input = interrupt(data)  ← 从这里继续
                    │    ...（继续执行）                    │
                    └─────────────────────────────────────┘
```

---

### 8.2 完整示例（转账审核）

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import END, START, StateGraph
from langgraph.types import Command, interrupt
from typing_extensions import TypedDict

class TransferState(TypedDict):
    recipient: str
    amount: int
    approved: bool
    final_status: str

def review_transfer(state: TransferState) -> dict:
    # interrupt：暂停执行，将数据传出，等待用户决策
    user_review = interrupt({
        "title": "转账审核",
        "pending": {"recipient": state["recipient"], "amount": state["amount"]},
        "instruction": "返回 True(批准) 或 dict(修改并批准)"
    })
    
    # interrupt 之后的代码，在 resume 时才继续执行
    approved = user_review if isinstance(user_review, bool) else user_review.get("approved", True)
    amount = state["amount"] if isinstance(user_review, bool) else user_review.get("amount", state["amount"])
    
    return {"approved": approved, "amount": amount}

def execute_transfer(state: TransferState) -> dict:
    if not state["approved"]:
        return {"final_status": "已取消"}
    return {"final_status": f"已转账 {state['amount']} 给 {state['recipient']}"}

# 构建图（必须配置 checkpointer！）
builder = StateGraph(TransferState)
builder.add_node("review_transfer", review_transfer)
builder.add_node("execute_transfer", execute_transfer)
builder.add_edge(START, "review_transfer")
builder.add_edge("review_transfer", "execute_transfer")
builder.add_edge("execute_transfer", END)
graph = builder.compile(checkpointer=InMemorySaver())

config = {"configurable": {"thread_id": "transfer-001"}}

# 第一次调用：图在 interrupt 处暂停
result = graph.invoke(
    {"recipient": "Alice", "amount": 100, "approved": False, "final_status": ""},
    config=config
)

# 从结果中取出要审核的数据
interrupt_data = result["__interrupt__"][0].value
print("等待审核的数据：", interrupt_data)

# 模拟用户决策（批准，但修改金额）
user_decision = {"approved": True, "amount": 80}

# 第二次调用：用 Command(resume=...) 传入决策，继续执行
final = graph.invoke(Command(resume=user_decision), config=config)
print("最终结果：", final["final_status"])
```

---

### 8.3 interrupt 使用注意事项

> ⚠️ **重要：幂等性问题！**

```
第二次调用 invoke(Command(resume=...)) 时：
  review_transfer 节点从函数起点重新执行
     ↓
  执行到 interrupt() → 用 resume 值继续
     ↓
  interrupt() 之前的代码 再次执行了！

如果 interrupt() 之前有：
  - 数据库写入  → 会重复写入 ❌
  - API 调用    → 会重复调用 ❌
  - 发送邮件    → 会重复发送 ❌

解决方案：
  1. 确保操作幂等性
  2. 将有副作用的操作移到 interrupt() 之后
  3. 将有副作用的操作放到单独的节点中
```

---

## 九、面试重点速查

### 🎯 高频考题

**1. LangGraph 是什么？和 LangChain 的区别？**
> LangGraph 是低级 Agent 编排框架，将工作流建模为图（节点+边+状态）。区别：LangGraph 原生支持**循环、状态机、持久化、并行**，LangChain 是线性链式调用，无法优雅处理需要循环/状态/回溯的复杂场景。

**2. State 有哪几种定义方式？推荐哪种？**
> TypedDict（推荐）、Pydantic BaseModel、dataclass。推荐 TypedDict，简洁且与 LangGraph 集成最佳。

**3. Reducer 是什么？默认行为是什么？**
> Reducer 定义节点输出的增量状态如何与全局状态合并。默认行为是**覆盖**（新值替换旧值）。`add_messages` 是追加，`operator.add` 对列表是拼接。通过 `Annotated[类型, reducer函数]` 语法指定。

**4. 为什么节点只能返回增量状态，不能返回整个 state？**
> LangGraph 底层会将节点输出作为增量状态与全局状态合并。返回整个 state 会导致：对于无 Reducer 的键出现合并异常；对于有 Reducer 的键，不属于当前节点的值也会被合并，产生数据错误。

**5. Checkpointer 解决了什么问题？有哪几种？**
> 解决：多次调用间保持上下文（短期记忆）+ 故障断点续传。三种：`InMemorySaver`（开发测试）、`SqliteSaver`（本地持久化）、`PostgresSaver`（生产分布式）。

**6. Pregel 超步（SuperStep）的执行逻辑？**
> Plan（确定本步执行哪些 Actor）→ Execute（并行执行）→ Update（更新 Channels）循环迭代。与传统 DAG 不同：节点在其订阅通道被更新时就会执行，可能**多次执行**（如菱形汇聚的末端节点）。

**7. thread_id 的作用？**
> 用于隔离不同会话的状态空间。同一 thread_id 的多次调用共享状态（实现短期记忆）；不同 thread_id 完全隔离。本质是 Checkpointer 中状态的查找 Key。

**8. interrupt 的工作原理？使用时有什么注意事项？**
> 在节点内调用 `interrupt(data)` 会暂停图执行，将 data 传出。用户审核后通过 `invoke(Command(resume=decision))` 恢复执行。注意：**interrupt 之前的代码在 resume 时会重新执行**，需保证幂等性或将有副作用操作移到 interrupt 之后。

**9. 条件边如何实现循环？**
> 通过 `add_conditional_edges` 添加路由函数，路由函数可以返回已执行过的节点名，从而构成循环。需配合 `recursion_limit` 防止死循环（默认25步）。

**10. stream 有哪些模式？各自的用途？**
> `values`（完整状态）/ `updates`（增量）/ `custom`（自定义数据）/ `messages`（LLM Token 流）/ `debug`（调试）/ 混合模式。生产常用：`messages`（打字机效果）+ `custom`（进度推送）。

---

### 📦 常用包和 API 速查

```python
# 核心导入
from langgraph.graph import StateGraph, START, END
from langgraph.constants import START, END
from langgraph.types import Command, interrupt, RetryPolicy, CachePolicy, StreamWriter, Send
from langgraph.runtime import Runtime
from langchain_core.runnables import RunnableConfig

# Checkpointer
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.sqlite import SqliteSaver

# 缓存
from langgraph.cache.memory import InMemoryCache

# 错误处理
from langgraph.errors import GraphRecursionError

# 常用 API
graph = StateGraph(state_schema=MyState, input_schema=..., output_schema=...)
graph.add_node("name", fn, cache_policy=..., retry_policy=...)
graph.add_edge(from_node, to_node)
graph.add_conditional_edges(from_node, router_fn, mapping)

compiled = graph.compile(checkpointer=..., cache=...)
compiled.invoke(input, config={"configurable": {"thread_id": "..."}}, context={...})
compiled.stream(input, stream_mode="messages")  # or values/updates/custom/debug/[list]
compiled.get_state(config=...)
compiled.get_state_history(config=...)
```

---

### 🗺️ LangGraph 核心知识图谱

```
                        LangGraph
                           │
          ┌────────────────┼────────────────┐
          │                │                │
        State            Node             Edge
          │                │                │
    TypedDict/          Python fn      普通边/条件边
    Pydantic/           state/          串行/并行
    dataclass           config/         循环结构
          │             runtime          │
    Reducer             │                │
    覆盖/追加/        缓存/重试        条件边路由
    自定义             │             recursion_limit
          │         stream_writer
          └──────────────┼────────────────┘
                         │
                   Checkpointer
                         │
               ┌─────────┴─────────┐
          InMemory              SQLite/Postgres
               │                   │
          短期记忆              断点续传
          thread_id隔离         生产持久化
                         │
                     Pregel 引擎
                         │
                    超步循环执行
                    Plan/Execute/Update
```
