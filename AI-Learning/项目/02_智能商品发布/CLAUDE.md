# 02_智能商品发布 — 商品标题分类器

> 基于 BERT 微调的商品标题多分类系统，含 AMP 混合精度、早停、断点续训、FastAPI 部署。

## 项目信息

- **类型**：微调系统（BERT 微调）
- **模型**：bert-base-chinese（本地加载）
- **框架**：PyTorch + HuggingFace + FastAPI
- **任务**：多分类（商品标题 → 类目）
- **数据**：TSV 格式（train/valid/test.txt）
- **训练技巧**：AMP 混合精度、Early Stopping、Checkpoint 断点续训

## 目录结构

```
02_智能商品发布/
├── CLAUDE.md                    ← 你在这里
├── 00_项目总览.md               ← Meta层+架构+4层映射+3种讲法
├── 01_系统架构与数据流.md       ← 四层架构+数据流Mermaid图
├── 02_核心模块设计.md           ← 6个模块含AMP/早停/Checkpoint
├── 03_问题与优化.md             ← 踩坑+优化+扩展
├── 04_面试表达.md               ← 30秒/2分钟/5分钟+STAR
├── 05_面试追问.md               ← 15个追问+标准答案
└── 详细参考/                    ← 旧版模块笔记
```

## 笔记阅读顺序

1. `00_项目总览.md` — 30秒了解项目全貌
2. `04_面试表达.md` — 直接背面试讲法
3. `02_核心模块设计.md` — 深入理解 AMP/早停/Checkpoint 等工程细节
4. `05_面试追问.md` — 准备追问答案
5. `03_问题与优化.md` — 了解改进方向

## 工程亮点

1. **标签固定映射**：sorted(set(labels)) + cast_column，保证训练/推理 id 一致
2. **动态 padding**：DataCollatorWithPadding，比静态 padding 省 40-60% 计算
3. **混合精度 AMP**：fp16 前向 + fp32 权重，GradScaler 防梯度下溢
4. **早停统一化**：loss 取负 / metric 直接比较，统一"越大越好"逻辑
5. **Checkpoint 完整状态**：model + optimizer + step + count + score + scaler
6. **配置管理**：pathlib 反推根目录，任意机器 clone 后无需改路径

## 关键词

BERT、微调、文本分类、多分类、AMP、Early Stopping、Checkpoint、FastAPI、动态padding、断点续训
