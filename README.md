# 🧠 AI Learning Vault

> 个人 AI 学习知识库 — 算法刷题 · 大模型笔记 · 项目实战
>
> Personal AI learning knowledge base — Algorithm practice · LLM notes · Project documentation

---

## 中文

### 📚 仓库简介

这是我的个人 AI 学习知识库，记录了从算法基础到大模型开发的完整学习路径。所有内容以 Markdown 格式整理，使用 Obsidian 管理，兼顾个人知识沉淀与云端备份。

**当前内容规模：** 150+ 道算法题解 · 19 个专题分类 · 2 个完整项目 · 4 篇大模型学习笔记

---

### 📂 内容导航

| 模块 | 说明 | 状态 |
|------|------|------|
| [算法](AI-Learning/算法/) | 150+ 道 LeetCode 题解，覆盖 19 个专题分类 | ✅ 持续更新 |
| [大模型](AI-Learning/大模型/) | Transformer、预训练、Hugging Face 生态、FastAPI 等笔记 | ✅ 持续更新 |
| [项目](AI-Learning/项目/) | AI 应用项目的完整开发文档（需求→训练→部署） | ✅ 持续更新 |

---

### 🧩 算法刷题笔记

基于 Python 的 LeetCode 题解，每道题包含：**思路拆解 → 完整代码 → 复杂度分析 → 关键注释**。

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

涵盖大语言模型从理论到工程的核心知识：

- **Transformer 架构** — 注意力机制、位置编码、编码器-解码器结构
- **预训练与微调** — 预训练范式、微调策略、训练技巧
- **Hugging Face 生态** — Transformers 库、数据集处理、模型部署
- **工程实践** — FastAPI + SQLAlchemy 后端开发笔记

---

### 🚀 项目实战

#### 00_AI 智评 — 电商评论情感分析

基于 BERT 的中文电商评论正/负情感分类项目，包含两个版本：

| 版本 | 方案 | 说明 |
|------|------|------|
| V1 | 手动搭建 | 自定义 `nn.Module` + BERT backbone，手动控制训练全流程 |
| V2 | HuggingFace 封装 | `AutoModelForSequenceClassification` 高阶 API 实现 |

> 完整记录了从数据预处理、模型训练、评估到预测部署的全流程文档。

#### 01_智能商品发布

基于 AI 的智能商品发布系统，项目文档持续完善中。

---

### 🛠️ 技术栈

`Python` · `PyTorch` · `Hugging Face Transformers` · `BERT` · `FastAPI` · `SQLAlchemy`

`VS Code` · `Obsidian` · `Git` · `GitHub`

---

### 📌 说明

- 所有内容仅供学习参考，部分笔记可能处于更新状态
- Markdown 文件可使用 VS Code、Obsidian 或任意编辑器直接打开
- 欢迎交流讨论，如有错误或建议请提 Issue

---

## English

### 📚 Overview

This is my personal AI learning knowledge base, documenting the full learning path from algorithm fundamentals to large model development. All content is organized in Markdown and managed with Obsidian.

**Current scale:** 150+ algorithm solutions · 19 topic categories · 2 projects · 4 LLM study notes

---

### 📂 Navigation

| Section | Description | Status |
|---------|-------------|--------|
| [Algorithms](AI-Learning/算法/) | 150+ LeetCode solutions across 19 topic categories | ✅ Active |
| [LLM Notes](AI-Learning/大模型/) | Transformer, pre-training, Hugging Face ecosystem, FastAPI | ✅ Active |
| [Projects](AI-Learning/项目/) | End-to-end AI project documentation (requirements → training → deployment) | ✅ Active |

---

### 🧩 Algorithm Practice

Python-based LeetCode solutions. Each entry includes: **approach breakdown → complete code → complexity analysis → key comments**.

<details>
<summary>📂 Topic Categories (19)</summary>

| Category | Coverage |
|----------|----------|
| Kadane's Algorithm | Max subarray, circular subarray |
| Dynamic Programming | 1D & multi-dimensional DP (climbing stairs, coin change, LIS, etc.) |
| Binary Search | Search insert position, find peak, rotated array |
| Binary Tree | Traversal, construction, path sum, LCA, BST iterator |
| Bit Manipulation | Single number, reverse bits |
| Two Pointers | Container with most water, 3Sum, palindrome check |
| Backtracking | Permutations, combinations, letter combinations |
| Graph | Number of islands, clone graph, surrounded regions |
| Sliding Window | Longest substring without repeating, minimum window substring |
| Linked List | Reverse, merge, LRU cache, reverse in k-group |
| Stack | Valid parentheses, min stack, basic calculator |
| Array & String | Best time to buy/sell stock, rotate array, zigzag conversion |
| More... | Hash table, heap, trie, math, matrix, divide & conquer, intervals |

</details>

---

### 🤖 LLM Study Notes

Core knowledge spanning theory to engineering for large language models:

- **Transformer Architecture** — Attention mechanisms, positional encoding, encoder-decoder structure
- **Pre-training & Fine-tuning** — Pre-training paradigms, fine-tuning strategies, training techniques
- **Hugging Face Ecosystem** — Transformers library, dataset processing, model deployment
- **Engineering Practice** — FastAPI + SQLAlchemy backend development notes

---

### 🚀 Projects

#### 00_AI Review — E-commerce Sentiment Analysis

BERT-based Chinese e-commerce review sentiment classification (positive/negative), with two implementations:

| Version | Approach | Description |
|---------|----------|-------------|
| V1 | Manual | Custom `nn.Module` + BERT backbone with full training control |
| V2 | HuggingFace API | High-level `AutoModelForSequenceClassification` implementation |

> Complete documentation covering data preprocessing, model training, evaluation, and prediction deployment.

#### 01_Smart Product Publishing

AI-powered smart product listing system. Documentation in progress.

---

### 🛠️ Tech Stack

`Python` · `PyTorch` · `Hugging Face Transformers` · `BERT` · `FastAPI` · `SQLAlchemy`

`VS Code` · `Obsidian` · `Git` · `GitHub`

---

### 📌 Notes

- All content is for learning reference only; some notes may be in progress
- Markdown files can be opened directly with VS Code, Obsidian, or any text editor
- Feel free to open an issue for corrections or suggestions

---
