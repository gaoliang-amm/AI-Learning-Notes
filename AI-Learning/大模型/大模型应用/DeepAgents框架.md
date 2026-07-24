---
aliases:
  - DeepAgents
  - 深度代理
  - 深度智能体
tags:
  - agent
  - multi-agent
  - deepagents
  - langchain
created: 2026-07-06
---

> 📚 来源：尚硅谷大模型项目之深度搜索
> 🎯 目标：理解 DeepAgents 在 LangChain 生态中的定位，掌握多智能体协作模式
> 🔗 关联笔记：[[大模型应用/LangChain|LangChain]]、[[大模型应用/LangGraph|LangGraph]]、[[大模型应用/Agent与工具调用|Agent与工具调用]]

---

## 一、定位：为什么需要 DeepAgents？

> 解决的问题：LangChain 能调用工具，LangGraph 能编排流程，但都没有内置"任务规划 + 子代理分工 + 长期记忆"——DeepAgents 补上这层。

### 1.1 三者关系

```
LangChain   →  做"动作"   →  LLM + 工具交互，单 Agent
LangGraph   →  管"流程"   →  图结构，支持循环/并行/持久化
DeepAgents  →  负责"组织"  →  内置规划器、子代理、文件系统、记忆
```

**类比**：LangChain 是工人，LangGraph 是工位流水线，DeepAgents 是项目经理。

### 1.2 何时选择

| 场景 | 选择 |
|:---|:---|
| 简单代理，快速原型 | LangChain |
| 复杂流程，需要精细控制 | LangGraph |
| 长期运行，自主规划，多步骤任务 | DeepAgents |

> 💡 **关键理解**：DeepAgents 建立在 LangChain 之上，使用 LangGraph 作为运行时。三者是层层递进，不是互斥。

### 1.3 两大驱动概念

| 概念 | 角色 | 核心 |
|:---|:---|:---|
| **Deep Agents（深度代理）** | 组织架构师 | 规划-执行-反馈-迭代闭环 |
| **Higher-Order Prompts（高阶提示）** | 认知规范师 | 教会模型如何思考 |

**传统提示 vs 高阶提示**：

| 类型 | 做什么 | 示例 |
|:---|:---|:---|
| 传统提示 | 告诉模型**做什么** | "分析这条评论，判断情绪" |
| 高阶提示 | 告诉模型**怎么想、按什么步骤** | "先逐句提取事实→判断情绪→按严重程度排序→给出建议→总结痛点" |

> 💡 高阶提示传递的是**思考框架与推理范式**，不只是结果指令。

---

## 二、四大核心能力（必须记住）

> 解决的问题：单 Agent 面对复杂任务时的四个致命弱点。

| 能力 | 解决什么 | 一句话 |
|:---|:---|:---|
| **智能规划** | 任务太复杂，不知从何下手 | 自带 todo 清单，边执行边调整 |
| **上下文管理** | 信息太多，上下文窗口爆了 | 文件系统外部化，按需读取 |
| **子代理协作** | 一个 Agent 干不完/干不好 | task 工具分工，上下文隔离 |
| **长期记忆** | 换个对话就忘了之前说的 | 跨线程持久化存储 |

### 2.1 智能规划：todo 机制详解

DeepAgents 内置 todo 规划机制，使代理能够：
- 将复杂任务拆分为离散、可执行的步骤
- 在执行过程中持续跟踪当前进度
- 根据新信息**动态调整**后续计划

**类比**：办生日派对 → 先拆解任务（确定时间地点、邀请朋友、买食材...），执行中发现蛋糕店关门 → 自动调整为线上订购

> 💡 **与传统工作流的区别**：不是死板地按预设步骤执行，而是先规划、再执行、边执行边调整。

### 2.2 上下文管理：文件系统工具

DeepAgents 提供文件系统与上下文外部化能力，核心工具：

| 工具 | 作用 |
|:---|:---|
| `ls` | 查看当前有哪些文件 |
| `read_file` | 精确读取需要的内容 |
| `write_file` | 把中间结果保存起来 |
| `edit_file` | 修改已有内容 |

**核心思想**：代理的"上下文大脑"里只保留当前最重要的信息，大内容存到外部文件，按需读取。

> 💡 这不只是简单的"读写本地文件"，更是一种**把大上下文外部化、按需取用**的工作方式。

**与 LangGraph 的联系**：
- 智能规划 ≈ LangGraph 的 State + 条件边，但 DeepAgents 封装成了自动 todo 机制
- 上下文管理 = LangGraph 没有的，DeepAgents 独有
- 子代理协作 ≈ LangGraph 的 subgraph，但 DeepAgents 用 `task` 工具简化了调用
- 长期记忆 = LangGraph 的 Store，DeepAgents 直接复用

---

## 三、快速上手（最小可运行示例）

> 解决的问题：如何创建第一个 Deep Agent。

```python
from langchain.chat_models import init_chat_model
from deepagents import create_deep_agent

# 1. 初始化模型
llm = init_chat_model(model="qwen-max", model_provider="openai")

# 2. 创建 Agent（核心函数）
agent = create_deep_agent(
    model=llm,
    tools=[internet_search],        # 工具列表
    subagents=[],                   # 子代理列表
    system_prompt="你是一位研究员"
)

# 3. 运行
result = agent.invoke({"messages": [{"role": "user", "content": "搜索宇树机器人新闻"}]})
print(result['messages'][-1].content)  # 取最后一条消息
```

**返回结构**：
```python
{
    "messages": [
        HumanMessage(content='用户提问'),
        AIMessage(content='', tool_calls=[...]),   # 调用工具
        ToolMessage(content='工具结果'),
        AIMessage(content='最终回复')               # ← 取这个
    ]
}
```

### 3.1 流式输出与解析

> 解决的问题：实时看到 Agent 的思考过程和工具调用，而不是等最终结果。

```python
stream = deep_agent.stream({"messages": [{"role": "user", "content": "搜索宇树机器人新闻"}]})

for chunk in stream:
    for node_name, state in chunk.items():
        if not state or "messages" not in state:
            continue
        messages = state["messages"]
        if messages and isinstance(messages, list):
            last_msg = messages[-1]
            
            if node_name == "model":                    # 模型节点
                if last_msg.tool_calls:                 # 决定调用工具/子代理
                    for tc in last_msg.tool_calls:
                        if tc['name'] == 'task':
                            print(f"[调用子智能体] {tc['args']['subagent_type']}")
                        else:
                            print(f"[调用工具] {tc['name']}, 参数: {tc['args']}")
                elif last_msg.content:                  # 最终回复
                    print(f"[最终回复] {last_msg.content}")
            
            elif node_name == "tools":                  # 工具执行结果
                print(f"[工具结果] {last_msg.content[:100]}...")
```

**四种 chunk 场景**：

| 场景 | 节点名 | 内容 |
|:---|:---|:---|
| A: Agent 前置处理 | `PatchToolCallsMiddleware.before_agent` | 格式化用户输入 |
| B: 模型决定调用工具 | `model` | `tool_calls` 不为空 |
| C: 工具/子代理执行完成 | `tools` | 工具返回结果 |
| D: 模型最终回复 | `model` | `content` 有值，`tool_calls` 为空 |

---

## 四、子代理与多智能体（重点）

> 解决的问题：单 Agent 的注意力会被多领域任务稀释，分而治之提升专业度。

### 4.1 什么时候用多智能体（三条铁律）

| 铁律 | 场景 |
|:---|:---|
| 问题极度开放 | 无标准答案，需要多方向探索 |
| 存在领域冲突 | 跨两个以上专业领域（医疗+法律） |
| 需要多方向并行 | 天然可拆分为独立子任务 |

**否则别用**——Token 消耗指数级增长，调试成噩梦。

### 4.2 两种架构模式

| 模式 | 核心逻辑 | 优点 | 缺点 |
|:---|:---|:---|:---|
| **层级工作流** | 中央集权，主脑分派 | 可控性强 | 单点故障 |
| **协作工作流** | 去中心化，专家会诊 | 灵活性高 | 容易失控 |

DeepAgents 默认是**层级模式**：主 Agent 通过 `task` 工具调度子 Agent。

### 4.3 子代理配置

**配置字段表**：

| 字段              | 类型                | 必填  | 说明                    | 继承规则        |
| :-------------- | :---------------- | :-- | :-------------------- | :---------- |
| `name`          | str               | ✅   | 唯一标识，调用时使用            | -           |
| `description`   | str               | ✅   | 职能描述，主 Agent 据此判断是否调用 | -           |
| `system_prompt` | str               | ✅   | 执行指令，子代理必须自定义        | 不继承主 Agent  |
| `tools`         | list              | 可选  | 工具列表                  | 默认继承主 Agent，指定时完全覆盖 |
| `model`         | str/BaseChatModel | 可选  | 模型                    | 默认继承主 Agent |
| `middleware`    | list              | 可选  | 中间件（日志、限流等）           | 不继承主 Agent  |
| `skills`        | list              | 可选  | 技能文件路径                | -           |

**配置示例**：

```python
weather_agent = {
    "name": "weather_helper",              # 唯一标识（必填）
    "description": "用于查询天气信息",       # 主 Agent 据此判断是否调用（必填）
    "system_prompt": "你是一个天气助手",     # 子 Agent 的指令
    "tools": [],                           # 工具列表（不继承主 Agent）
    "model": llm                           # 模型（默认继承主 Agent）
}

main_agent = create_deep_agent(
    model=llm,
    subagents=[weather_agent],             # 注册子代理
    system_prompt="你是管家，调度助手解决问题"
)
```

**关键理解**：
- `description` 是主 Agent 决策的依据——写得越具体，调度越准
- 子代理上下文**默认隔离**：独立 Prompt、独立工具、独立记忆
- 目的：专注、安全、模块化

**什么时候用子代理**：
- 多步骤任务会让主 Agent 的上下文变得杂乱
- 有需要"专业技能/专属工具"的环节（如股票分析：基本面用财务工具、技术面用 K 线工具）
- 需要不同模型能力的任务（多模态）
- 想让主 Agent 专注于高层协调

**什么时候不用子代理**：
- 任务简单，一步就能干完
- 需要中间信息连贯，不能拆（如"读文章→总结"，拆了会丢上下文）
- 运营费用超过收益时

### 4.4 CompiledSubAgent（兼容已有 Agent）

> 解决的问题：已有的 LangGraph 图或 LangChain Agent 如何挂载为 DeepAgents 子代理？

**核心要求**：子代理必须是「带有 `messages` 键的状态图」。

**写法一：兼容 LangGraph 编译图**

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, END
from langgraph.graph.message import add_messages
from deepagents import CompiledSubAgent
from langchain_core.messages import AIMessage

# 定义 State（必须包含 messages）
class SubState(TypedDict):
    messages: Annotated[list, add_messages]

# 定义节点
def processing_node(state: SubState):
    last_msg = state["messages"][-1]
    return {"messages": [AIMessage(content=f"处理完成: {last_msg.content}")]}

# 构建并编译图
workflow = StateGraph(SubState)
workflow.add_node("worker", processing_node)
workflow.set_entry_point("worker")
workflow.add_edge("worker", END)
compiled_graph = workflow.compile()

# 封装为 CompiledSubAgent
sub_agent = CompiledSubAgent(
    name="complex_worker",
    description="处理复杂业务逻辑",
    runnable=compiled_graph
)

main_agent = create_deep_agent(model=llm, subagents=[sub_agent])
```

**写法二：兼容 LangChain 单智能体**

```python
from langchain.agents import create_agent
from deepagents import CompiledSubAgent

# 创建 LangChain Agent
agent = create_agent(model=llm, tools=[get_weather])

# 封装为 CompiledSubAgent
custom_subagent = CompiledSubAgent(
    name="subagent",
    description="查询天气信息",
    runnable=agent
)

main_agent = create_deep_agent(model=llm, subagents=[custom_subagent])
```

### 4.5 异步执行（astream）

> 解决的问题：高并发服务、批量处理、非阻塞主线程场景。

```python
import asyncio

async def query_agent(query):
    async for chunk in main_agent.astream({"messages": [{"role": "user", "content": query}]}):
        for node_name, state in chunk.items():
            if not state or "messages" not in state:
                continue
            messages = state["messages"]
            if messages and isinstance(messages, list):
                last_msg = messages[-1]
                if node_name == "model" and last_msg.content:
                    print(f"[回复] {last_msg.content}")

# 并发执行多个查询
async def batch_run():
    await asyncio.gather(
        query_agent("北京天气怎么样？"),
        query_agent("100 + 256 等于多少？"),
        query_agent("将你好翻译成英文")
    )

asyncio.run(batch_run())
```

**何时用 `astream()` 而非 `stream()`**：
- **高并发服务**：FastAPI/Starlette 做接口时，`astream()` + 异步能同时处理成百上千个用户请求，不会因单个请求阻塞整个服务
- **批量处理任务**：需要同时调用智能体处理多个查询，`astream()` 并发执行耗时 ≈ 最长单个任务，比同步 `stream()` 串行快几倍
- **非阻塞主线程**：GUI 程序（PyQt/Tkinter）、定时任务中调用智能体，`astream()` 不会让界面卡死/定时任务中断
- **MCP 工具返回异步结果时**（必须用，否则报错 `StructuredTool does not support sync invocation`）

### 4.6 与 LangGraph subgraph 的区别

| 维度 | DeepAgents 子代理 | LangGraph subgraph |
|:---|:---|:---|
| 定义方式 | 字典配置 | StateGraph 编程 |
| 调度方式 | `task` 工具自动调度 | 手动在边中调用 |
| 上下文 | 默认隔离 | 需手动管理 |
| 适用场景 | 快速搭建 | 精细控制 |

---

## 五、人机交互 (HITL)

> 解决的问题：高危操作（删库、发邮件）需要人工确认才能执行。

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.types import Command

agent = create_deep_agent(
    model=llm,
    tools=[delete_database, delete_file, select_data],
    interrupt_on={
        "delete_database": True,           # 需要审批
        "delete_file": True,
        "select_data": False               # 无需审批
    },
    checkpointer=MemorySaver()             # 必须：保存中断状态
)

# 第一次调用：触发中断
thread_config = {"configurable": {"thread_id": "safe_1"}}
result = agent.invoke({"messages": [...]}, config=thread_config)

# 检查中断并审批
if result.get("__interrupt__"):
    action_requests = result["__interrupt__"][0].value['action_requests']
    
    decisions = [
        {"type": "approve"},               # 同意
        {"type": "reject"},                # 拒绝
        # {"type": "edit", "edited_action": {...}}  # 编辑参数
    ]
    
    # 第二次调用：恢复执行（必须相同 thread_id）
    result = agent.invoke(
        Command(resume={"decisions": decisions}),
        config=thread_config
    )
```

**核心要点**：
- `checkpointer` 必须配置，否则无法中断恢复
- 恢复时必须使用相同的 `thread_id`
- 审批类型：approve（同意）、reject（拒绝）、edit（改参数后执行）

**edit 审批（修改参数后执行）**：
```python
decisions = []
for action in action_requests:
    if action["name"] == "delete_database":
        decisions.append({
            "type": "edit",
            "edited_action": {
                "name": action["name"],
                "args": {"table_name": "test_users"}   # 改成测试表
            }
        })
```

**stream 模式下的 HITL**：
```python
# 第一次流式执行：触发中断
for chunk in deep_agent.stream({"messages": [...]}, config=thread_config):
    if "__interrupt__" in chunk:
        interrupts = chunk["__interrupt__"]
        break

# 审批后恢复
for chunk in deep_agent.stream(
    Command(resume={"decisions": decisions}),
    config=thread_config
):
    print("resume chunk =>", chunk)
```

---

## 六、后端存储 (Backends)

> 解决的问题：Agent 生成的文件存哪里？如何跨会话共享数据？

**核心机制**：
- **被动触发**：Backend 仅在 Agent 主动调用文件操作工具（`write_file`、`read_file` 等）时才被激活。思考过程、对话上下文等临时状态仅存内存（State），不会自动写入 Backend
- **路径映射**：Agent 操作的文件基于"虚拟路径"（如 `/report.txt`），Backend 按规则映射到实际物理存储（本地硬盘/Redis/内存）

**存储行为对照**：

| 行为 | Backend 是否存储 | 存储位置 |
|:---|:---|:---|
| Agent 说"你好" | 否 | 仅在当前对话内存 (State) |
| Agent 思考过程 | 否 | 仅在当前对话内存 (State) |
| Agent 调用 `write_file("a.txt", "内容")` | 是 | Backend (硬盘/数据库) |

| 后端 | 存储 | 场景 |
|:---|:---|:---|
| **StateBackend**（默认） | 内存 | 临时，会话结束即销毁 |
| **FilesystemBackend** | 本地硬盘 | 开发调试 |
| **StoreBackend** | 数据库 KV | 生产环境，跨 Agent 共享 |
| **CompositeBackend** | 混合 | 按路径路由，最佳实践 |

**FilesystemBackend（本地文件）**：
```python
from deepagents.backends import FilesystemBackend

backend = FilesystemBackend(root_dir=workspace_dir, virtual_mode=True)  # virtual_mode 限制访问范围
agent = create_deep_agent(model=llm, backend=backend)
```

**StoreBackend（数据库/内存）**：
```python
from deepagents.backends import StoreBackend
from langgraph.store.memory import InMemoryStore

store = InMemoryStore()
agent = create_deep_agent(
    model=llm,
    store=store,
    backend=StoreBackend(namespace=lambda ctx: ("filesystem",))
)

# 跨线程共享：Thread A 写入，Thread B 读取（同一个 store）
```

**CompositeBackend（混合存储，推荐）**：
```python
from deepagents.backends import CompositeBackend, FilesystemBackend, StoreBackend

composite_backend = CompositeBackend(
    default=fs_backend,                  # 默认存本地
    routes={"/store/": store_backend}    # /store/ 前缀存数据库
)

agent = create_deep_agent(model=llm, store=store, backend=composite_backend)
# 普通文件 → 本地磁盘
# /store/memory.txt → 数据库（跨线程共享）
```

**与 LangGraph 的联系**：DeepAgents 的 StoreBackend 直接复用 LangGraph 的 Store 机制。

---

## 七、权限管理 (Permissions)

> 解决的问题：限制 Agent 对文件的读/写访问，做路径级黑白名单。

**生效范围**：内置文件工具（`ls`、`read_file`、`write_file` 等），自定义工具不生效。

**匹配规则**：从上到下顺序匹配，命中第一条就生效；无规则命中 → 默认允许所有读写。

**示例：全局只读**：
```python
from deepagents import FilesystemPermission

agent = create_deep_agent(
    model=llm,
    backend=backend,
    permissions=[
        FilesystemPermission(operations=["write"], paths=["/**"], mode="deny")
    ]
)
```

**示例：目录隔离（仅允许访问工作区）**：
```python
agent = create_deep_agent(
    model=llm,
    backend=backend,
    permissions=[
        FilesystemPermission(operations=["read", "write"], paths=["/agent_workspace/**"], mode="allow"),
        FilesystemPermission(operations=["read", "write"], paths=["/**"], mode="deny"),  # 其他全部拒绝
    ]
)
```

---

## 八、技能扩展 (Skills)

> 解决的问题：如何让 Agent 按需加载领域知识，而不把所有信息塞进 system_prompt？

**核心机制：渐进式披露**
- 启动时：只读所有技能的元数据（name + description），轻量
- 运行时：任务匹配触发条件后，才加载 SKILL.md 详细内容

**目录结构**：
```
skill-xxx/
├── SKILL.md              # 必选：元数据(YAML) + 指令(Markdown)
├── requirements.txt      # 可选：依赖声明
└── scripts/              # 可选：辅助脚本
```

**SKILL.md 示例**：
```markdown
---
name: code-reviewer
description: 当用户请求代码审查时使用此技能
---
# Code Reviewer
## 角色
你是资深架构师。
## 审查标准
1. 安全性：SQL 注入、硬编码密钥
2. 性能：重复计算、无效循环
```

```python
agent = create_deep_agent(
    model=llm,
    backend=backend,
    skills=["skills"]    # 指向技能目录
)
```

---

## 九、上下文工程 (Context Engineering)

> 解决的问题：给 Agent 正确的信息结构，而不是堆砌更多提示词。

**核心区分**：输入上下文决定 Agent「是谁、会什么」，运行时上下文决定「这次任务用什么数据」。

### 9.1 输入上下文（Agent 创建时加载，塑造能力边界）

| 类型 | 内容 | 加载时机 | 类比 |
|:---|:---|:---|:---|
| **system_prompt** | 基础身份、行为准则 | 创建时一次性写入 | 员工入职培训 |
| **memory** | 长期规则、用户偏好、项目约定 | 每次对话始终加载 | 公司规章制度 |
| **skills** | 领域专业知识、操作指令 | 按需加载（匹配触发条件） | 岗位专业手册 |

**Memory vs Skills 决策**：
- Memory：**每次都可能用**的通用规则 → `memory=["/project/AGENTS.md"]`
  - 例："默认用中文回复"、"代码风格遵循 PEP8"
- Skills：**某类任务才用**的专业能力 → `skills=["skills"]`
  - 例："如何写祝福语"、"代码审查流程"

> 💡 输入上下文在 `create_deep_agent()` 时配置，运行中不可变。

### 9.2 运行时上下文（每次调用传入，提供动态数据）

> 不写进 prompt，不占上下文窗口，由工具内部按需读取。

```python
from dataclasses import dataclass
from langchain_core.tools import tool, ToolRuntime

# 1. 定义上下文结构
@dataclass
class Context:
    user_id: str
    api_key: str

# 2. 工具内部按需读取（不经过 LLM）
@tool
def fetch_user_data(query: str, runtime: ToolRuntime[Context]) -> str:
    user_id = runtime.context.user_id   # 工具内部读取，LLM 看不到
    return f"用户 {user_id} 的数据..."

# 3. 声明 Agent 时绑定 schema
agent = create_deep_agent(model=llm, tools=[fetch_user_data], context_schema=Context)

# 4. 每次调用传入不同值
result = agent.invoke(
    {"messages": [...]},
    context=Context(user_id="user-123", api_key="sk-xxx")
)
```

**两类上下文对比**：

| 维度 | 输入上下文 | 运行时上下文 |
|:---|:---|:---|
| 何时配置 | `create_deep_agent()` 时 | 每次 `invoke()` / `stream()` 时 |
| 谁能读 | LLM（直接看到 prompt） | 工具函数（通过 `ToolRuntime`） |
| 是否占上下文窗口 | 是 | 否 |
| 典型内容 | 身份、规则、专业指令 | user_id、API Key、会话标识 |
| 变更频率 | 低（几乎不变） | 高（每次调用不同） |

---

## 十、实战要点：旅游规划多智能体

> 不贴完整代码，只记关键设计决策。

### 架构

```
用户输入 → 主智能体（解析需求、调度全局）
              ├── map_agent      → 高德 Skill（景点/路线）
              ├── ticket_agent   → 12306 MCP（车次/票价）
              └── summary_agent  → 汇总生成最终方案
```

### 关键设计决策

1. **主 Agent 不干活，只调度** — 解析用户需求后，分发给专业子 Agent
2. **子 Agent 各司其职** — map 负责地理，ticket 负责交通，summary 负责汇总
3. **外部能力接入** — 地图用 Skill（高德），车票用 MCP（12306）
4. **异步流式输出** — MCP 工具返回异步结果，必须用 `astream()` 而非 `stream()`

### 与已有知识的联系

- **高德 Skill** — [[大模型应用/Agent与工具调用|工具调用]] + Skills 机制（渐进式加载）
- **12306 MCP** — [[大模型应用/Agent与工具调用#五、MCP（Model Context Protocol）|MCP 协议]]（独立标准，非 LangGraph 专属）
- **子代理调度** — DeepAgents 的 `task` 工具，参见本文 [[#四、子代理与多智能体（重点）|第四节]]
- **结果持久化** — [[#六、后端存储 (Backends)|CompositeBackend]]（本地 + Store 混合路由）

---

## 十一、面试速查

**Q1: DeepAgents 和 LangChain/LangGraph 什么关系？**
> 层层递进：LangChain 做动作 → LangGraph 管流程 → DeepAgents 负责组织。DeepAgents 建立在前两者之上。

**Q2: 四大核心能力？**
> 智能规划（todo 机制）、上下文管理（文件系统外部化）、子代理协作（task 工具）、长期记忆（跨线程持久化）。

**Q3: 什么时候用多智能体？**
> 三条铁律：问题极度开放 / 存在领域冲突 / 需要多方向并行。否则别用——Token 爆炸 + 调试噩梦。

**Q4: 子代理上下文隔离指什么？**
> 独立 Prompt、独立工具、独立记忆。目的：专注、安全、模块化。

**Q5: Skills 的渐进式披露？**
> 启动时只读元数据（name + description），匹配触发条件后才加载详细指令。避免无关信息占用上下文。

**Q6: HITL 的三个关键配置？**
> `interrupt_on`（哪些工具需要审批）、`checkpointer`（必须）、相同 `thread_id`（恢复时必须）。

**Q7: Memory 和 Skills 的区别？**
> Memory 始终加载（长期规则），Skills 按需加载（专业能力）。"默认说中文"放 Memory，"如何写祝福语"放 Skills。
