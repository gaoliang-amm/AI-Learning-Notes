---
aliases:
  - Agent
  - AI Agent
  - AI智能体
  - 工具调用
  - Function Calling
  - MCP
tags:
  - Agent
  - 大模型
  - 工具调用
  - Function Calling
  - MCP
  - ReAct
created: 2026-06-03
---

# Agent 与工具调用

> AI Agent 是以 LLM 为核心推理引擎，能够自主规划、使用工具、与环境交互并完成复杂任务的智能体系统。工具调用（Tool/Function Calling）是 Agent 能力的基础。

---

## 一、什么是 AI Agent

### Agent 的本质

```
传统 LLM: 用户提问 → LLM 生成回答（一次性）

AI Agent: 用户提出目标 → Agent 自主规划 → 循环执行 → 达成目标
                         ↑                    ↓
                         └── 观察结果 ←── 调用工具
```

### Agent 与 Chatbot 的区别

| 维度 | Chatbot | Agent |
|------|---------|-------|
| **交互模式** | 一问一答 | 自主规划、多步执行 |
| **能力边界** | 仅文本生成 | 可调用工具、访问外部系统 |
| **推理方式** | 单次推理 | 规划 → 执行 → 观察 → 反思 循环 |
| **记忆** | 仅上下文窗口 | 短期记忆 + 长期记忆 |
| **目标** | 回答问题 | 完成复杂任务 |

### Agent 的核心组成

```
┌──────────────────────────────────────────┐
│                 AI Agent                  │
│                                          │
│  ┌──────────┐  ┌──────────┐  ┌────────┐ │
│  │  规划     │  │  记忆     │  │ 工具   │ │
│  │ Planning │  │ Memory   │  │ Tools  │ │
│  └────┬─────┘  └────┬─────┘  └───┬────┘ │
│       │              │            │      │
│       └──────────┬───┘────────────┘      │
│                  ↓                       │
│         ┌──────────────┐                 │
│         │   LLM 大脑    │                 │
│         │  (推理引擎)   │                 │
│         └──────────────┘                 │
└──────────────────────────────────────────┘
```

---

## 二、Agent 核心能力

### 2.1 规划（Planning）

Agent 将复杂任务分解为可执行的步骤。

#### 规划策略

| 策略 | 原理 | 适用场景 |
|------|------|---------|
| **ReAct** | 推理(Reason) + 行动(Act) 交替执行 | **通用首选** |
| **Plan-and-Execute** | 先制定完整计划，再逐步执行 | 复杂多步任务 |
| **Reflexion** | 执行后反思，改进策略 | 需要迭代优化 |
| **Tree of Thoughts** | 探索多条推理路径，选择最优 | 需要深度推理 |
| **LATS** | 蒙特卡洛树搜索 + LLM | 复杂决策问题 |

#### ReAct 模式详解

ReAct 是最经典的 Agent 推理范式：

```
用户: "帮我查一下北京明天的天气，如果是雨天就提醒我带伞"

Thought: 用户需要查询北京明天的天气，然后根据结果决定是否提醒
Action: call weather_api(city="北京", date="明天")
Observation: 北京明天天气：小雨，15-22℃
Thought: 明天是雨天，需要提醒用户带伞
Action: send_notification("明天北京有小雨，记得带伞哦！")
Observation: 通知发送成功
Thought: 任务完成
Answer: 北京明天有小雨，气温15-22℃，我已经设置了提醒，记得带伞！
```

#### Plan-and-Execute 模式

```
用户: "帮我写一篇关于 RAG 的技术博客"

Plan:
  1. 调研 RAG 最新技术动态
  2. 整理核心概念和架构
  3. 编写博客大纲
  4. 逐章节撰写内容
  5. 润色和校对
  6. 生成最终文档

Execute:
  Step 1: search("RAG 最新进展 2024") → 获取信息
  Step 2: 基于信息整理概念 → 输出结构化知识
  Step 3: 生成大纲 → 用户确认
  ...逐步完成
```

### 2.2 记忆（Memory）

| 记忆类型 | 作用 | 实现方式 |
|---------|------|---------|
| **短期记忆** | 当前对话上下文 | 上下文窗口 / 对话历史 |
| **长期记忆** | 跨会话的知识 | 向量数据库 / 外部存储 |
| **工作记忆** | 当前任务的中间状态 | Scratchpad / 状态变量 |
| **情景记忆** | 过去的经历和经验 | 经验库 / 案例检索 |

#### 记忆管理策略

```python
# 记忆存储与检索
class AgentMemory:
    def __init__(self):
        self.short_term = []       # 对话历史
        self.long_term = VectorDB()  # 长期记忆

    def store(self, observation):
        """存储重要信息到长期记忆"""
        if self.is_important(observation):
            embedding = embed(observation)
            self.long_term.add(embedding, observation)

    def recall(self, query, top_k=5):
        """检索相关记忆"""
        # 先查短期记忆
        short_results = self.search_short_term(query)
        # 再查长期记忆
        long_results = self.long_term.search(embed(query), top_k)
        return self.merge(short_results, long_results)
```

### 2.3 工具使用（Tool Use）

Agent 通过调用外部工具扩展能力边界。

#### 常见工具类型

| 类别 | 工具示例 | 用途 |
|------|---------|------|
| **搜索** | Google、Bing、Tavily | 获取实时信息 |
| **代码执行** | Python REPL、沙箱 | 计算、数据处理 |
| **文件操作** | 读写文件、PDF 解析 | 文档处理 |
| **API 调用** | 天气、股票、邮件 | 获取/操作外部服务 |
| **数据库** | SQL、向量数据库 | 数据查询 |
| **浏览器** | Playwright、Selenium | 网页交互 |
| **知识库** | RAG 检索 | 私域知识查询 |
| **其他 Agent** | 子 Agent | 任务委派 |

---

## 三、Function Calling 机制

Function Calling 是 LLM 调用外部工具的标准接口。

### 3.1 工作流程

```
1. 定义工具 → 描述工具名称、功能、参数（JSON Schema）
2. 用户提问 → LLM 分析是否需要调用工具
3. LLM 输出 → 结构化的工具调用请求（函数名 + 参数）
4. 执行工具 → 应用层执行实际调用
5. 返回结果 → 将工具返回值送回 LLM
6. 生成回答 → LLM 基于工具结果生成最终回答
```

### 3.2 工具定义格式（OpenAI 风格）

```json
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "获取指定城市的天气信息",
    "parameters": {
      "type": "object",
      "properties": {
        "city": {
          "type": "string",
          "description": "城市名称，如：北京、上海"
        },
        "date": {
          "type": "string",
          "description": "日期，格式 YYYY-MM-DD，默认今天"
        }
      },
      "required": ["city"]
    }
  }
}
```

### 3.3 调用示例（OpenAI API）

```python
import openai

# 定义工具
tools = [
    {
        "type": "function",
        "function": {
            "name": "search_knowledge",
            "description": "搜索知识库获取相关信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词"}
                },
                "required": ["query"]
            }
        }
    }
]

# 第一轮：LLM 决定调用工具
response = openai.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "什么是 RAG？"}],
    tools=tools,
    tool_choice="auto"
)

# LLM 返回工具调用请求
tool_call = response.choices[0].message.tool_calls[0]
# → function: search_knowledge(query="RAG 检索增强生成")

# 第二轮：执行工具并返回结果
result = search_knowledge("RAG 检索增强生成")  # 实际执行

# 第三轮：将结果送回 LLM
messages = [
    {"role": "user", "content": "什么是 RAG？"},
    response.choices[0].message,  # assistant 的工具调用消息
    {
        "role": "tool",
        "tool_call_id": tool_call.id,
        "content": str(result)
    }
]

final_response = openai.chat.completions.create(
    model="gpt-4",
    messages=messages
)
# → 最终回答
```

### 3.4 各模型 Function Calling 支持

| 模型 | 支持程度 | 特点 |
|------|---------|------|
| **GPT-4 / GPT-4o** | 完善 | 原生支持，多工具并行调用 |
| **Claude 3.5** | 完善 | 原生支持，Tool Use |
| **Gemini** | 完善 | 原生支持 |
| **Qwen2.5** | 完善 | 开源模型中支持较好 |
| **DeepSeek-V3** | 完善 | 支持 Function Calling |
| **LLaMA 3.1** | 有限 | 需要格式引导 |
| **Mistral** | 完善 | 原生支持 |

---

## 四、Agent 架构模式

### 4.1 单 Agent 架构

```
用户 → Agent (LLM + Tools) → 结果
```

- 最简单的架构
- 适合任务明确、工具不多的场景
- 局限：上下文窗口有限，工具过多时选择困难

### 4.2 Multi-Agent 架构

```
用户 → 主 Agent (Orchestrator)
           ├── 子 Agent A (搜索专家)
           ├── 子 Agent B (代码专家)
           └── 子 Agent C (写作专家)
                    ↓
              汇总结果 → 最终回答
```

#### Multi-Agent 协作模式

| 模式 | 说明 | 适用场景 |
|------|------|---------|
| **主从模式** | 一个主 Agent 分配任务给子 Agent | 任务可清晰分解 |
| **对等模式** | 多个 Agent 平等协作、互相讨论 | 需要多角度分析 |
| **流水线模式** | Agent 按顺序处理，上一个的输出是下一个的输入 | 有明确处理步骤 |
| **辩论模式** | 多个 Agent 从不同角度辩论，得出最佳结论 | 需要深度思考 |
| **投票模式** | 多个 Agent 独立完成，投票选最优 | 需要高可靠性 |

#### Multi-Agent 框架

| 框架 | 特点 |
|------|------|
| **AutoGen** | 微软开源，支持多 Agent 对话协作 |
| **CrewAI** | 角色扮演式多 Agent 协作 |
| **LangGraph** | LangChain 出品，基于图的 Agent 编排 |
| **Swarm** | OpenAI 出品，轻量级多 Agent |
| **MetaGPT** | 模拟软件公司组织结构 |
| **Camel** | 双 Agent 角色扮演协作 |

### 4.3 Agentic Workflow

将 Agent 嵌入业务工作流：

```
触发条件（定时/事件/用户请求）
   ↓
Agent 接收任务
   ↓
规划执行步骤
   ↓
循环执行:
  ├─ 调用工具 A
  ├─ 分析结果
  ├─ 决定下一步
  ├─ 调用工具 B
  └─ ...
   ↓
生成报告 / 执行动作
   ↓
通知用户 / 记录日志
```

---

## 五、MCP（Model Context Protocol）

MCP 是 Anthropic 提出的开放协议，标准化 LLM 与外部工具/数据源的连接方式。

### 5.1 MCP 解决的问题

```
之前: 每个 LLM 应用 × 每个工具 = M×N 个集成
现在: 所有 LLM 应用 ↔ MCP 协议 ↔ 所有工具 = M+N 个集成
```

### 5.2 MCP 架构

```
┌─────────────┐     MCP 协议      ┌─────────────┐
│  MCP Client  │ ←───────────────→ │  MCP Server  │
│  (LLM 应用)  │                   │  (工具提供方) │
└─────────────┘                   └──────┬──────┘
                                         │
                                    ┌────┴────┐
                                    │ 实际资源  │
                                    │ DB/API/FS│
                                    └─────────┘
```

### 5.3 MCP 核心概念

| 概念 | 说明 | 示例 |
|------|------|------|
| **Tools** | 可被 LLM 调用的函数 | `search(query)`, `send_email(to, body)` |
| **Resources** | 可被读取的数据源 | 文件内容、数据库记录 |
| **Prompts** | 预定义的提示模板 | 代码审查模板、翻译模板 |
| **Sampling** | 服务器请求 LLM 生成 | 工具内部需要 LLM 推理 |

### 5.4 MCP Server 示例

```python
# Python MCP Server 示例
from mcp.server import Server
from mcp.types import Tool, TextContent

server = Server("my-tools")

@server.tool()
async def search_web(query: str) -> str:
    """搜索网页获取信息"""
    results = await web_search(query)
    return format_results(results)

@server.tool()
async def read_file(path: str) -> str:
    """读取本地文件内容"""
    with open(path, 'r') as f:
        return f.read()

@server.resource("config://app")
async def get_config() -> str:
    """获取应用配置"""
    return json.dumps(config)

# 启动 MCP Server
server.run(transport="stdio")
```

### 5.5 MCP Client 调用

```python
from mcp.client import ClientSession, StdioServerParameters

# 连接 MCP Server
async with ClientSession(
    StdioServerParameters(command="python", args=["my_server.py"])
) as session:
    # 列出可用工具
    tools = await session.list_tools()

    # 调用工具
    result = await session.call_tool(
        "search_web",
        arguments={"query": "什么是 RAG"}
    )
```

### 5.6 MCP 生态

| 类别 | MCP Server | 功能 |
|------|-----------|------|
| **文件系统** | filesystem | 读写本地文件 |
| **数据库** | postgres, sqlite | SQL 查询 |
| **搜索引擎** | brave-search, tavily | 网络搜索 |
| **版本控制** | github, gitlab | Git 操作 |
| **协作工具** | slack, notion | 消息/文档 |
| **开发工具** | docker, k8s | 容器管理 |
| **知识库** | memory, obsidian | 知识管理 |

---

## 六、主流 Agent 框架

### 6.1 LangChain / LangGraph

```python
from langchain.agents import AgentExecutor, create_react_agent
from langchain_openai import ChatOpenAI
from langchain.tools import Tool

# 定义工具
tools = [
    Tool(name="Search", func=search, description="搜索信息"),
    Tool(name="Calculate", func=calculate, description="数学计算"),
]

# 创建 Agent
llm = ChatOpenAI(model="gpt-4")
agent = create_react_agent(llm, tools, prompt)
executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# 执行
result = executor.invoke({"input": "北京到上海的直线距离是多少？"})
```

### 6.2 LangGraph（状态图 Agent）

```python
from langgraph.graph import StateGraph, END
from typing import TypedDict

class AgentState(TypedDict):
    messages: list
    next_step: str

def think(state: AgentState) -> AgentState:
    """LLM 思考下一步"""
    response = llm.invoke(state["messages"])
    # 解析是否需要调用工具
    return {**state, "messages": state["messages"] + [response]}

def use_tool(state: AgentState) -> AgentState:
    """执行工具调用"""
    tool_call = extract_tool_call(state["messages"][-1])
    result = execute_tool(tool_call)
    return {**state, "messages": state["messages"] + [result]}

def should_continue(state: AgentState) -> str:
    """决定是否继续"""
    if needs_tool_call(state["messages"][-1]):
        return "use_tool"
    return END

# 构建状态图
graph = StateGraph(AgentState)
graph.add_node("think", think)
graph.add_node("use_tool", use_tool)
graph.add_edge("think", should_continue)
graph.add_edge("use_tool", "think")

agent = graph.compile()
result = agent.invoke({"messages": [user_message], "next_step": "think"})
```

### 6.3 CrewAI

```python
from crewai import Agent, Task, Crew

# 定义角色
researcher = Agent(
    role="研究员",
    goal="深入调研给定主题",
    backstory="你是一位资深技术研究员",
    tools=[search_tool, read_tool],
    llm=llm
)

writer = Agent(
    role="技术作家",
    goal="将研究成果写成通俗易懂的文章",
    backstory="你是一位优秀的技术博客作者",
    llm=llm
)

# 定义任务
research_task = Task(
    description="调研 RAG 技术的最新进展",
    expected_output="详细的调研报告",
    agent=researcher
)

writing_task = Task(
    description="基于调研报告撰写技术博客",
    expected_output="一篇 2000 字的技术博客",
    agent=writer,
    context=[research_task]  # 依赖调研任务的结果
)

# 创建团队
crew = Crew(
    agents=[researcher, writer],
    tasks=[research_task, writing_task],
    verbose=True
)

# 执行
result = crew.kickoff()
```

### 6.4 OpenAI Assistants API

```python
from openai import OpenAI
client = OpenAI()

# 创建 Assistant
assistant = client.beta.assistants.create(
    name="数据分析师",
    instructions="你是一位专业的数据分析师，善于使用 Python 进行数据分析。",
    model="gpt-4",
    tools=[
        {"type": "code_interpreter"},   # 代码执行
        {"type": "file_search"},        # 文件搜索
        {
            "type": "function",
            "function": {
                "name": "query_database",
                "description": "查询数据库",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "sql": {"type": "string", "description": "SQL 查询语句"}
                    },
                    "required": ["sql"]
                }
            }
        }
    ]
)

# 创建对话线程
thread = client.beta.threads.create()

# 发送消息
client.beta.threads.messages.create(
    thread_id=thread.id,
    role="user",
    content="分析上个月的销售数据，找出 Top 10 产品"
)

# 运行 Assistant
run = client.beta.threads.runs.create(
    thread_id=thread.id,
    assistant_id=assistant.id
)

# 等待完成并获取结果
```

---

## 七、Agent 设计模式

### 7.1 ReAct 模式

```
循环:
  Thought: 分析当前状态，决定下一步
  Action: 调用工具 (tool_name, arguments)
  Observation: 获取工具返回结果
  → 直到任务完成
```

### 7.2 Reflection 模式

```
执行任务 → 生成结果
   ↓
自我反思: "结果质量如何？有什么问题？"
   ↓
改进策略 → 重新执行 → 更好的结果
   ↓
（可选）外部评估: 评分/反馈
```

### 7.3 Tool-Use 模式

```
用户请求
   ↓
LLM 分析: 需要哪些工具？
   ↓
选择工具 + 生成参数
   ↓
执行工具 → 获取结果
   ↓
（可能循环多次）
   ↓
综合结果 → 最终回答
```

### 7.4 Planning 模式

```
用户目标
   ↓
LLM 生成执行计划 (Plan)
   ↓
逐步执行 Plan 中的每个 Step
   ↓
每步执行后检查:
  ├─ 成功 → 继续下一步
  ├─ 失败 → 重新规划
  └─ 偏离 → 调整计划
   ↓
所有步骤完成 → 汇总结果
```

---

## 八、Agent 评估

### 评估维度

| 维度 | 指标 | 说明 |
|------|------|------|
| **任务完成率** | Success Rate | 能否正确完成目标任务 |
| **效率** | Steps to Complete | 完成任务需要的步骤数 |
| **工具使用** | Tool Accuracy | 工具选择和参数生成的准确性 |
| **推理质量** | Reasoning Quality | 推理过程是否合理 |
| **鲁棒性** | Robustness | 面对异常情况的处理能力 |
| **安全性** | Safety | 是否产生危险操作 |
| **成本** | Token Cost | 完成任务消耗的 Token 数 |

### 评估基准

| 基准 | 评估内容 | 说明 |
|------|---------|------|
| **SWE-bench** | 软件工程任务 | 修复 GitHub Issue |
| **WebArena** | 网页交互任务 | 完成网页操作 |
| **GAIA** | 通用 AI 助手 | 多步骤真实任务 |
| **AgentBench** | 综合 Agent 能力 | 多环境评估 |
| **ToolBench** | 工具使用能力 | 16000+ 真实 API |
| **τ-bench** | 对话式 Agent | 多轮对话任务 |

---

## 九、安全与风险

### Agent 安全风险

| 风险 | 说明 | 防御措施 |
|------|------|---------|
| **Prompt 注入** | 恶意输入劫持 Agent 行为 | 输入过滤、指令层级隔离 |
| **越权操作** | Agent 执行超出预期的操作 | 最小权限原则、操作确认 |
| **数据泄露** | Agent 访问/暴露敏感数据 | 访问控制、数据脱敏 |
| **无限循环** | Agent 陷入死循环 | 最大步骤限制、超时机制 |
| **工具滥用** | 频繁/错误调用工具 | 速率限制、成本控制 |
| **幻觉传播** | 错误推理导致错误操作 | 人类审批、结果验证 |

### 安全最佳实践

```
1. 最小权限: Agent 只拥有完成任务所需的最小权限
2. 人类审批: 关键操作（支付、删除、发送）需人类确认
3. 沙箱执行: 代码执行在隔离环境中
4. 日志审计: 记录所有工具调用和决策过程
5. 成本限制: 设置 Token 和 API 调用上限
6. 输入验证: 过滤和验证所有外部输入
```

---

## 十、Agent 应用场景

### 典型应用场景

| 场景 | Agent 能力 | 示例 |
|------|-----------|------|
| **代码开发** | 编写、调试、重构代码 | GitHub Copilot Workspace |
| **数据分析** | 查询、分析、可视化数据 | 数据分析 Agent |
| **客服支持** | 理解问题、查询知识库、回复用户 | 智能客服 |
| **内容创作** | 调研、写作、编辑 | 博客写作 Agent |
| **自动化运维** | 监控告警、故障排查、自动修复 | DevOps Agent |
| **个人助理** | 日程管理、邮件处理、信息整理 | AI 秘书 |
| **研究助手** | 文献检索、摘要、对比分析 | 学术研究 Agent |

---

## 十一、发展趋势

1. **Agent 原生模型** — 模型从预训练阶段就针对 Agent 场景优化
2. **多模态 Agent** — 能看、能听、能说、能操作的全能 Agent
3. **自主进化** — Agent 从经验中学习，持续改进策略
4. **Agent 社会** — 大量 Agent 协作、竞争、形成社会结构
5. **标准化协议** — MCP 等协议推动工具生态标准化
6. **Agent 安全框架** — 专门的安全评估和防护机制

---

## 相关笔记

- [[大模型应用/RAG技术全解|RAG技术全解]] — Agent 常用的知识检索工具
- [[AI缩写术语表]] — 相关缩写解释
- [[大模型应用/Transformer|Transformer]] — LLM 底层架构
- [[大模型应用/预训练|预训练]] — Agent 大脑的基础
