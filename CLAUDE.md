

本文件为 Claude Code (claude.ai/code) 在此仓库中工作时提供指导。

## 仓库概述

这是一个 **Obsidian 个人知识库**（非代码项目），用于 AI 学习。内容以 Markdown 笔记形式组织，使用 Obsidian 管理，Git 版本控制，托管于 GitHub。

## 目录结构

```
AI-Learning/
├── 算法/          # 150+ 道 LeetCode Python 题解，19 个专题分类
├── 大模型/        # LLM 笔记：Transformer、预训练、微调、部署、RAG、Agent
│   ├── 00. 大模型知识体系导航.md   # 全局导读，串联全部笔记
│   ├── 01. LLM基础与GPT-2.md      # Ch1-2: LLM概念、GPT-2架构与实现
│   ├── 02. 强化学习基础.md         # Ch3-4: MDP、策略梯度、Actor-Critic
│   ├── 03. PPO与RLHF对齐.md       # Ch5-9: PPO裁剪、重要性采样、RLHF流程
│   ├── 04. GRPO与DeepSeek-R1.md   # Ch10-12: DPO/GRPO/DAPO、DeepSeek-R1复刻
│   ├── 05. 多模态模型.md           # Ch13-15: ViT、CLIP、ClipCap、Qwen2.5-VL
│   ├── 06. 扩散模型与文生图.md     # Ch16-20: DDPM、条件扩散、DALL-E2
│   └── ...                        # 其他已有笔记（LangChain、RAG、Agent等）
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

### 大模型系列笔记规范（00-06号）

基于《大语言模型、强化学习和多模态实践》PDF 教程整理，共 7 个文件：

| 文件 | PDF章节 | 核心内容 |
|------|--------|---------|
| 00. 大模型知识体系导航 | 全局 | 三条演进主线、贯穿概念、学习顺序、常见困惑 |
| 01. LLM基础与GPT-2 | Ch1-2 | NTP、Decoder-Only Transformer、GPT-2架构 |
| 02. 强化学习基础 | Ch3-4 | MDP、策略梯度、REINFORCE、Actor-Critic、GAE |
| 03. PPO与RLHF对齐 | Ch5-9 | PPO裁剪目标、重要性采样、GRPO原理、RLHF三阶段 |
| 04. GRPO与DeepSeek-R1 | Ch10-12 | DPO理论与实战、InstructGPT复刻、DAPO、CountDown Task |
| 05. 多模态模型 | Ch13-15 | ViT、CLIP对比学习、ClipCap图生文、Qwen2.5-VL |
| 06. 扩散模型与文生图 | Ch16-20 | DDPM、U-Net、Classifier-Free Guidance、DALL-E2三模型 |

**交叉引用规则**：所有笔记在"相关笔记"部分链接到其他 6 个笔记 + 00号导航，使用 `[[00. 大模型知识体系导航]]` 格式。

**直觉解释风格**：重要概念附带通俗类比（如"PPO裁剪=方向盘限位器"、"GRPO组优势=全班考试看排名"），帮助理解。

## Git 工作流

- 默认分支：`main`
- commit message 使用简洁英文
- 不自动 commit 或 push，除非明确要求
- 提交前先展示变更摘要

## 注意事项

- `.obsidian/` 目录为 Obsidian 配置，不要修改
- 部分 `.md` 文件包含 Mermaid 图表（需 Obsidian Mermaid 插件渲染）
- 在 `AI-Learning/大模型/` 下新建笔记时，遵循现有 frontmatter 格式（`aliases`、`tags`、`created`）
- 算法题解文件命名规则：`{题号}. {题目名}.md`
