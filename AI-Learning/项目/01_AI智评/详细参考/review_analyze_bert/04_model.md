# 🧠 model.py — 模型定义

定义了 `ReviewAnalyzeModel`，核心思路：**用 BERT 提取文本特征，再接一个线性层做二分类**。

---

## 模型架构图

```
输入文本（已编码）
        ↓
  ┌─────────────┐
  │  BERT 模型   │  （从预训练权重加载，提取语义特征）
  └─────────────┘
        ↓
   [CLS] 向量             shape: (batch_size, 768)
        ↓
  ┌─────────────┐
  │  Linear 层   │  768 → 1
  └─────────────┘
        ↓
   输出标量值             shape: (batch_size,)
        ↓
   Sigmoid → 概率值 [0,1]
        ↓
   > 0.5 → 正面，≤ 0.5 → 负面
```

---

## 代码详解

### 类定义

```python
class ReviewAnalyzeModel(nn.Module):
    def __init__(self):
        super(ReviewAnalyzeModel, self).__init__()
        # 加载预训练 BERT
        self.bert = AutoModel.from_pretrained(MODEL_NAME_OR_PATH)
        # 线性分类头：768维 → 1维
        self.classifier = nn.Linear(self.bert.config.hidden_size, 1)
```

> 💡 `self.bert.config.hidden_size`：BERT-base 的隐藏层维度是 **768**，BERT-large 是 1024。代码自动适配，不写死。

---

### 前向传播 `forward()`

```python
def forward(self, input_ids, attention_mask, token_type_ids=None):
    # 1. BERT 编码
    outputs = self.bert(
        input_ids=input_ids,
        attention_mask=attention_mask,
        token_type_ids=token_type_ids
    )
    # 2. 取 [CLS] 位置的向量
    cls_hidden_state = outputs.pooler_output    # shape: (batch_size, 768)
    # 3. 线性层得到标量
    output = self.classifier(cls_hidden_state)  # shape: (batch_size, 1)
    
    return output.squeeze(-1)                   # shape: (batch_size,)
```

#### 为什么取 `pooler_output`（CLS 向量）？

BERT 输出每个 token 的向量，但 `[CLS]` 是句子开头的特殊标记，BERT 在预训练时专门让它**汇总整句语义**，因此做句子分类时用它最合适。

```
[CLS] 质 量 很 好 [SEP]  →  BERT  →  768维向量（每个token都有）
  ↑
取这个 → pooler_output
```

---

## 训练 vs 推理的区别

| 阶段 | 有无梯度 | 调用方式 |
|------|----------|----------|
| 训练 | ✅ 有 | `model.train()` |
| 推理 | ❌ 无（`torch.no_grad()`） | `model.eval()` |

---

## 🔗 关联笔记

- [[01_config]] → `MODEL_NAME_OR_PATH`
- [[05_train]] → 实例化此模型并训练
- [[07_predict]] → 推理时调用此模型
- [[08_新手必读_核心概念]] → BERT 是什么？CLS 向量？
