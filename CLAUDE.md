

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 仓库概述

这是一个 **Obsidian 个人知识库**（非代码项目），用于 AI 学习。内容以 Markdown 笔记形式组织，使用 Obsidian 管理，Git 版本控制，托管于 GitHub。

## 目录结构

```
AI-Learning/
├── 算法/          # 150+ 道 LeetCode Python 题解，19 个专题分类
│   ├── 00. 数据结构与算法基础.md    # 八大排序、查找、DP、回溯、贪心
│   └── ...                        # 按专题分类的题解
├── 大模型/        # LLM 学习笔记（4 个子目录 + 导航）
│   ├── 00. 大模型知识体系导航.md   # 全局导读，学习路线，所有笔记索引
│   ├── 强化学习与多模态/          # PDF《大语言模型、强化学习和多模态实践》
│   │   ├── 01. LLM基础与GPT-2.md
│   │   ├── 02. 强化学习基础.md
│   │   ├── 03. PPO与RLHF对齐.md
│   │   ├── 04. GRPO与DeepSeek-R1.md
│   │   ├── 05. 多模态模型.md
│   │   └── 06. 扩散模型与文生图.md
│   ├── 基础课程/                  # 尚硅谷基础课程笔记
│   │   ├── 01. 数学基础.md        # 高等数学、线性代数、概率论
│   │   ├── 02. 机器学习基础.md    # ML流程、线性回归/逻辑回归/SVM/决策树/集成学习
│   │   ├── 03. 深度学习基础.md    # PyTorch、神经网络、CNN、优化器、正则化
│   │   └── 04. NLP基础.md         # 分词、Word2Vec、RNN/LSTM/GRU、Seq2Seq、Attention
│   ├── 大模型应用/                # Transformer、预训练、LangChain、Agent、RAG
│   │   ├── Transformer.md
│   │   ├── 预训练.md
│   │   ├── LLM技术全景笔记.md
│   │   ├── LangChain.md
│   │   ├── LangGraph.md
│   │   ├── Agent与工具调用.md
│   │   └── RAG技术全解.md
│   └── 工程实践/                  # 工具链与工程开发
│       ├── Docker.md
│       ├── Git.md
│       ├── Linux.md
│       ├── Shell.md
│       ├── numpy与pandas.md
│       ├── FastAPI_SQLAlchemy_笔记.md
│       └── HuggingFace生态.md
├── 基础概念/      # AI 缩写术语表
├── 模板/          # 笔记模板
└── 项目/          # AI 项目文档（需求 → 训练 → 部署）
```

## 内容规范

- 笔记语言为**中文**，技术术语保留英文（模型名、代码、文件路径）
- 算法笔记格式统一：**思路拆解 → 完整代码 → 复杂度分析 → 关键注释**
- 大模型笔记使用 Obsidian 特性：`[[wikilinks]]` 双向链接、frontmatter（`aliases`/`tags`）、Mermaid 图
- 算法题解使用 Python；大模型笔记包含 PyTorch/Transformers 代码片段
- 笔记之间大量交叉引用——创建新文件前先检查已有的 `[[链接]]` 保持一致性
- 跨目录引用使用相对路径格式：`[[子目录/文件名|显示名]]`

### 强化学习与多模态系列（00-06号）

基于《大语言模型、强化学习和多模态实践》PDF 教程整理，共 7 个文件：

| 文件 | PDF章节 | 核心内容 |
|------|--------|---------|
| 01. LLM基础与GPT-2 | Ch1-2 | NTP、Decoder-Only Transformer、GPT-2架构 |
| 02. 强化学习基础 | Ch3-4 | MDP、策略梯度、REINFORCE、Actor-Critic、GAE |
| 03. PPO与RLHF对齐 | Ch5-9 | PPO裁剪目标、重要性采样、GRPO原理、RLHF三阶段 |
| 04. GRPO与DeepSeek-R1 | Ch10-12 | DPO理论与实战、InstructGPT复刻、DAPO、CountDown Task |
| 05. 多模态模型 | Ch13-15 | ViT、CLIP对比学习、ClipCap图生文、Qwen2.5-VL |
| 06. 扩散模型与文生图 | Ch16-20 | DDPM、U-Net、Classifier-Free Guidance、DALL-E2三模型 |

**直觉解释风格**：重要概念附带通俗类比（如"PPO裁剪=方向盘限位器"、"GRPO组优势=全班考试看排名"）。

### 基础课程笔记规范

基于尚硅谷课程整理，面试导向 + 知识体系串联：

| 文件 | 对应课程 | 核心内容 |
|------|---------|---------|
| 01. 数学基础 | 数学基础 | 极限、导数、梯度、矩阵运算、矩阵求导、概率分布、贝叶斯、MLE |
| 02. 机器学习基础 | 机器学习 | 特征工程、损失函数、评估指标、线性/逻辑回归、SVM、决策树、集成学习 |
| 03. 深度学习基础 | 深度学习 | PyTorch、Tensor、神经网络、激活函数、反向传播、优化器、CNN |
| 04. NLP基础 | NLP | 分词、Word2Vec、RNN/LSTM/GRU、Seq2Seq、Attention、Transformer |

**笔记风格**：每篇包含知识体系串联 + 面试高频FAQ + 背诵版总结。全面覆盖源文档所有算法和概念。

## Git 工作流

- 默认分支：`main`
- commit message 使用简洁英文
- 不自动 commit 或 push，除非明确要求
- 提交前先展示变更摘要

## 注意事项

- `.obsidian/` 目录为 Obsidian 配置，不要修改
- 部分 `.md` 文件包含 Mermaid 图表（需 Obsidian Mermaid 插件渲染）
- 新建笔记时遵循现有 frontmatter 格式（`aliases`、`tags`、`created`）
- 算法题解文件命名规则：`{题号}. {题目名}.md`
- 工具类笔记（Docker/Git/Linux/Shell/numpy与pandas）定位为**速查参考**，精简不啰嗦
