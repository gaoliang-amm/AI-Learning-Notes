# 01_AI智评 — 电商评论情感分析

> 基于 BERT 的中文电商评论正/负情感二分类，有两个版本（手动搭建 + HuggingFace AutoModel）。

## 项目信息

- **类型**：微调系统（BERT 微调）
- **模型**：bert-base-chinese
- **框架**：PyTorch + HuggingFace transformers + datasets
- **任务**：二分类（正面/负面）
- **数据**：电商评论 CSV（文本+标签）

## 目录结构

```
01_AI智评/
├── CLAUDE.md                    ← 你在这里
├── 00_项目总览.md               ← Meta层+架构+4层映射+3种讲法
├── 01_系统架构与数据流.md       ← 架构+数据流+V1/V2差异
├── 02_核心模块设计.md           ← 6个模块：作用+为什么+对比+代码
├── 03_问题与优化.md             ← 踩坑+优化路径+扩展
├── 04_面试表达.md               ← 30秒/2分钟/5分钟+STAR
├── 05_面试追问.md               ← 15个追问+标准答案
└── 详细参考/                    ← 旧版详细笔记（V1+V2+讲项目）
```

## 笔记阅读顺序

1. `00_项目总览.md` — 30秒了解项目全貌
2. `04_面试表达.md` — 直接背面试讲法
3. `02_核心模块设计.md` — 深入理解每个模块
4. `05_面试追问.md` — 准备追问答案
5. `03_问题与优化.md` — 了解踩坑和改进方向

## 核心差异（V1 vs V2）

| V1（手动） | V2（AutoModel） |
|-----------|----------------|
| 自己写 model.py | 不需要 |
| BCEWithLogitsLoss | CrossEntropyLoss（内置） |
| sigmoid(outputs) | softmax(logits)[:,1] |
| torch.save(state_dict) | save_pretrained |

## 关键词

BERT、微调、情感分析、二分类、HuggingFace、动态padding、BCEWithLogitsLoss、CrossEntropyLoss
