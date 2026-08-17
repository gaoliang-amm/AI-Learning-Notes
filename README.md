# 🧠 AI Learning Vault

> 个人 AI 学习知识库 — 算法刷题 · 大模型笔记 · 工具参考 · 项目实战
>
> Personal AI learning knowledge base — Algorithm practice · LLM notes · Tool references · Projects

---

## 中文

### 📚 仓库简介

这是我的个人 AI 学习知识库，记录了从算法基础到大模型开发的完整学习路径。所有内容以 Markdown 格式整理，使用 Obsidian 管理，兼顾个人知识沉淀与云端备份。

**当前内容规模：** 150+ 道算法题解 · 19 个专题分类 · 4 篇基础课程 · 6 篇 PDF 系列 · 7 篇应用/工程笔记 · 5 篇工具参考 · 5 个完整项目

---

### 📂 内容导航

| 模块 | 说明 | 状态 |
|------|------|------|
| [算法](AI-Learning/算法/) | 150+ 道 LeetCode 题解，覆盖 19 个专题分类 + 数据结构基础 | ✅ 持续更新 |
| [大模型-基础课程](AI-Learning/大模型/基础课程/) | 数学基础、机器学习、深度学习、NLP | ✅ 已完成 |
| [大模型-强化学习与多模态](AI-Learning/大模型/强化学习与多模态/) | LLM基础、强化学习、PPO/RLHF、GRPO、多模态、扩散模型 | ✅ 持续更新 |
| [大模型-应用](AI-Learning/大模型/大模型应用/) | Transformer、预训练、LangChain、LangGraph、Agent、RAG | ✅ 已完成 |
| [工程实践](AI-Learning/大模型/工程实践/) | Docker、Git、Linux、Shell、numpy/pandas、FastAPI、HuggingFace | ✅ 已完成 |
| [项目](AI-Learning/项目/) | AI 应用项目的完整开发文档（需求→训练→部署） | ✅ 持续更新 |

---

### 🧩 算法刷题笔记

基于 Python 的 LeetCode 题解，每道题包含：**思路拆解 → 完整代码 → 复杂度分析 → 关键注释**。

另有 [数据结构与算法基础](AI-Learning/算法/00.%20数据结构与算法基础.md) 笔记，覆盖八大排序、二分查找、动态规划、回溯、贪心等核心算法。

<details>
<summary>📂 专题分类（19 个）</summary>

| 分类 | 包含内容 |
|------|---------|
| Kadane 算法 | 最大子数组和、环形子数组 |
| 动态规划 | 一维 DP、多维 DP（爬楼梯、零钱兑换、最长递增子序列等） |
| 二分查找 | 搜索插入位置、寻找峰值、旋转数组 |
| 二叉树 | 遍历、构造、路径、LCA、BST 迭代器 |
| 位运算 | 只出现一次的数字、颠倒二进制位 |
| 双指针 | 盛水容器、三数之和、回文验证 |
| 回溯 | 全排列、组合、电话号码 |
| 图 | 岛屿数量、克隆图、被围绕的区域 |
| 滑动窗口 | 无重复字符最长子串、最小覆盖子串 |
| 链表 | 反转、合并、LRU 缓存、K 个一组翻转 |
| 栈 | 有效括号、最小栈、基本计算器 |
| 数组与字符串 | 买卖股票、轮转数组、Z 字形变换 |
| 更多... | 哈希表、堆、字典树、数学、矩阵、分治、区间 |

</details>

---

### 🤖 大模型学习笔记

按学习路线组织，从基础到应用再到工程实践：

**基础课程**（面试导向 + 知识体系串联）：
- [数学基础](AI-Learning/大模型/基础课程/01.%20数学基础.md) — 极限、导数、梯度、矩阵运算、概率论、贝叶斯、MLE
- [机器学习基础](AI-Learning/大模型/基础课程/02.%20机器学习基础.md) — 特征工程、线性/逻辑回归、SVM、决策树、集成学习
- [深度学习基础](AI-Learning/大模型/基础课程/03.%20深度学习基础.md) — PyTorch、神经网络、CNN、反向传播、优化器
- [NLP基础](AI-Learning/大模型/基础课程/04.%20NLP基础.md) — 分词、Word2Vec、RNN/LSTM/GRU、Attention、Transformer

**强化学习与多模态**（PDF 系列，可编辑）：
- LLM基础与GPT-2、强化学习基础、PPO与RLHF、GRPO与DeepSeek-R1、多模态模型、扩散模型与文生图

**大模型应用**：
- [Transformer 讲解稿](AI-Learning/大模型/大模型应用/Transformer.md) — 架构详解，面试八股
- [预训练](AI-Learning/大模型/大模型应用/预训练.md) — GPT/BERT/T5 预训练与微调
- [LangChain](AI-Learning/大模型/大模型应用/LangChain.md) · [LangGraph](AI-Learning/大模型/大模型应用/LangGraph.md) · [Agent与工具调用](AI-Learning/大模型/大模型应用/Agent与工具调用.md) · [RAG技术全解](AI-Learning/大模型/大模型应用/RAG技术全解.md)

---

### 🛠️ 工程实践与工具参考

工具类笔记定位为**速查参考**，精简实用：

| 工具 | 笔记 | 说明 |
|------|------|------|
| Docker | [Docker.md](AI-Learning/大模型/工程实践/Docker.md) | 容器技术、镜像管理、Dockerfile、docker-compose |
| Git | [Git.md](AI-Learning/大模型/工程实践/Git.md) | 版本控制、分支策略、冲突解决 |
| Linux | [Linux.md](AI-Learning/大模型/工程实践/Linux.md) | 文件操作、文本处理、权限、vim |
| Shell | [Shell.md](AI-Learning/大模型/工程实践/Shell.md) | 变量、条件、循环、函数、grep/sed/awk |
| numpy/pandas | [numpy与pandas.md](AI-Learning/大模型/工程实践/numpy与pandas.md) | ndarray、DataFrame、数据清洗、聚合 |
| FastAPI | [FastAPI_SQLAlchemy.md](AI-Learning/大模型/工程实践/FastAPI_SQLAlchemy_笔记.md) | Web 后端开发 |
| HuggingFace | [HuggingFace生态.md](AI-Learning/大模型/工程实践/HuggingFace生态.md) | Transformers 库、模型部署 |

---

### 🚀 项目实战

#### 01_AI 智评 — 电商评论情感分析

基于 BERT 的中文电商评论正/负情感分类项目，包含两个版本：

| 版本 | 方案 | 说明 |
|------|------|------|
| V1 | 手动搭建 | 自定义 `nn.Module` + BERT backbone，手动控制训练全流程 |
| V2 | HuggingFace 封装 | `AutoModelForSequenceClassification` 高阶 API 实现 |

> 完整记录了从数据预处理、模型训练、评估到预测部署的全流程文档。

#### 02_智能商品发布

基于 AI 的智能商品发布系统，项目文档持续完善中。

#### 03_掌柜智库 — 企业私有知识库问答

基于 RAG 架构 + LangGraph 工作流编排的企业级私有知识库智能问答系统。

| 特性 | 说明 |
|------|------|
| 核心设计 | 多路检索 + RRF 融合 + Rerank 精排 |
| 技术栈 | LangGraph + RAGFlow + Tavily + SSE |
| 业务场景 | 企业文档问答、知识管理、流式交互 |

> 工程化代码 5500+ 行，实现从文档接入、知识库构建到流式答案生成的全链路生产级能力。

#### 04_深度搜索 — 多智能体深度搜索

基于 DeepAgents 框架的多智能体深度搜索系统，采用「1 主 + 3 专」架构。

| 特性 | 说明 |
|------|------|
| 核心设计 | 主智能体统筹调度 + 专家子智能体并行协作 |
| 技术栈 | DeepAgents + Tavily + RAGFlow + WebSocket |
| 业务场景 | 复杂信息处理、多轮迭代搜索、研究报告生成 |

> 通过「搜索→阅读→反思→再搜索」多轮迭代，实现广覆盖、高精准的深度研究。

#### 05_电商智能客服 — LLM 路由 + 流程编排

基于 LLM 路由 + YAML 流程编排的电商智能客服系统，支持任务流程、知识问答、闲聊三种对话模式。

| 特性 | 说明 |
|------|------|
| 核心设计 | 三轨分离 + 白名单校验 + 澄清兜底 |
| 技术栈 | FastAPI + LangChain + 通义千问 + SQLAlchemy |
| 业务场景 | 查订单、查物流、退款申请、商品咨询、闲聊 |

> 包含完整的架构设计、核心模块详解、面试表达和追问准备。

---

### 🛠️ 技术栈

`Python` · `PyTorch` · `Hugging Face Transformers` · `BERT` · `LangChain` · `FastAPI`

`Docker` · `Git` · `Linux` · `NumPy` · `Pandas`

`VS Code` · `Obsidian` · `GitHub`

---

### 📌 说明

- 所有内容仅供学习参考，部分笔记可能处于更新状态
- Markdown 文件可使用 VS Code、Obsidian 或任意编辑器直接打开
- 笔记之间通过 Obsidian `[[wikilinks]]` 双向链接串联
- 欢迎交流讨论，如有错误或建议请提 Issue

---

## English

### 📚 Overview

This is my personal AI learning knowledge base, documenting the full learning path from algorithm fundamentals to large model development. All content is organized in Markdown and managed with Obsidian.

**Current scale:** 150+ algorithm solutions · 19 topics · 4 basic course notes · 6 PDF series · 7 application/engineering notes · 5 tool references · 5 projects

---

### 📂 Navigation

| Section | Description | Status |
|---------|-------------|--------|
| [Algorithms](AI-Learning/算法/) | 150+ LeetCode solutions, 19 topics + data structures basics | ✅ Active |
| [Basic Courses](AI-Learning/大模型/基础课程/) | Math, ML, Deep Learning, NLP | ✅ Complete |
| [RL & Multimodal](AI-Learning/大模型/强化学习与多模态/) | LLM, RL, PPO/RLHF, GRPO, Multimodal, Diffusion | ✅ Active |
| [LLM Applications](AI-Learning/大模型/大模型应用/) | Transformer, Pre-training, LangChain, Agent, RAG | ✅ Complete |
| [Engineering](AI-Learning/大模型/工程实践/) | Docker, Git, Linux, Shell, numpy/pandas, FastAPI, HuggingFace | ✅ Complete |
| [Projects](AI-Learning/项目/) | End-to-end AI project documentation | ✅ Active |

---

### 🚀 Projects

#### 01_AI Sentiment — E-commerce Review Sentiment Analysis

BERT-based Chinese e-commerce review sentiment classification with two versions:

| Version | Approach | Description |
|---------|----------|-------------|
| V1 | Manual | Custom `nn.Module` + BERT backbone |
| V2 | HuggingFace | `AutoModelForSequenceClassification` high-level API |

> Complete documentation from data preprocessing, model training, evaluation to deployment.

#### 02_Smart Product Publishing

AI-powered intelligent product publishing system, documentation in progress.

#### 03_Store Advisor — Enterprise Private Knowledge Base QA

Enterprise-grade private knowledge base QA system based on RAG + LangGraph workflow orchestration.

| Feature | Description |
|---------|-------------|
| Core Design | Multi-path retrieval + RRF fusion + Rerank |
| Tech Stack | LangGraph + RAGFlow + Tavily + SSE |
| Scenarios | Enterprise document QA, knowledge management |

> 5500+ lines of production code, full-chain capability from document ingestion to streaming answer generation.

#### 04_Deep Search — Multi-Agent Deep Search

Multi-agent deep search system based on DeepAgents framework with "1 Main + 3 Expert" architecture.

| Feature | Description |
|---------|-------------|
| Core Design | Main agent orchestration + Expert agents parallel collaboration |
| Tech Stack | DeepAgents + Tavily + RAGFlow + WebSocket |
| Scenarios | Complex information processing, multi-round search |

> Multi-round iteration via "search→read→reflect→re-search" for comprehensive and accurate research.

#### 05_E-commerce Customer Service — LLM Routing + Flow Orchestration

E-commerce customer service system based on LLM routing + YAML flow orchestration.

| Feature | Description |
|---------|-------------|
| Core Design | Three-track separation + Whitelist validation + Clarification fallback |
| Tech Stack | FastAPI + LangChain + Tongyi Qianwen + SQLAlchemy |
| Scenarios | Order tracking, logistics, refund, product consultation |

> Complete architecture design, core module analysis, interview expression and Q&A preparation.

---

### 🛠️ Tech Stack

`Python` · `PyTorch` · `Hugging Face Transformers` · `BERT` · `LangChain` · `FastAPI`

`Docker` · `Git` · `Linux` · `NumPy` · `Pandas`

`VS Code` · `Obsidian` · `GitHub`

---

### 📌 Notes

- All content is for learning reference only; some notes may be in progress
- Markdown files can be opened directly with VS Code, Obsidian, or any text editor
- Notes are cross-linked via Obsidian `[[wikilinks]]`
- Feel free to open an issue for corrections or suggestions

---
