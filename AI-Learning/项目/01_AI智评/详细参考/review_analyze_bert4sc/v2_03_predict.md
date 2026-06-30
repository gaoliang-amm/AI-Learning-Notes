# 🔮 v2 predict.py — 预测（logits + softmax）

本版预测逻辑有一处关键变化：模型输出从标量变成了 `logits`，概率转换从 `sigmoid` 变成了 `softmax`。

---

## `predict_batch` 对比

**第一版**（自定义模型，输出标量）：

```python
outputs = model(**inputs)           # shape: (batch_size,)  ← 一个数
batch_probs = torch.sigmoid(outputs) # 转为概率
return batch_probs.tolist()
```

**本版**（`AutoModelForSequenceClassification`，输出 logits）：

```python
outputs = model(**inputs)                          # outputs 是对象
batch_probs = torch.softmax(outputs.logits, dim=-1)  # outputs.logits shape: (batch_size, 2)
return batch_probs[:, 1].tolist()                  # 取第 1 列 = 正类概率
```

---

## 为什么输出变成了 2 列？

`AutoModelForSequenceClassification` 默认做**多分类**，输出每个类别的得分（logits）：

```
batch_size=4, num_labels=2

outputs.logits:
[[-1.2,  2.1],   ← 样本1：负类得分-1.2，正类得分2.1  → 正面
 [ 0.8, -0.5],   ← 样本2：负类得分0.8，正类得分-0.5  → 负面
 [-0.3,  1.7],   ← 样本3
 [ 1.1, -0.9]]   ← 样本4
```

经过 `softmax` 转成概率：

```
softmax 后：
[[0.03, 0.97],   ← 正面概率 97%
 [0.82, 0.18],   ← 正面概率 18%（即负面）
 [0.14, 0.86],
 [0.88, 0.12]]

取 [:, 1] → [0.97, 0.18, 0.86, 0.12]  ← 正类概率列表
```

---

## sigmoid vs softmax 的区别

| | `sigmoid` | `softmax` |
|--|-----------|-----------|
| 适用 | 二分类（输出1个值） | 多分类（输出N个值） |
| 公式 | `1 / (1 + e^-x)` | `e^xi / Σe^xj` |
| 各类概率之和 | 不一定等于1 | **等于1** |
| 本项目 | 第一版用 | 本版用 |

---

## 模型加载方式的变化

第一版：

```python
model = ReviewAnalyzeModel().to(device)           # 先建模型
model.load_state_dict(torch.load(MODELS_DIR / BEST_MODEL))  # 再加载权重
```

本版：

```python
model = AutoModelForSequenceClassification.from_pretrained(MODELS_DIR).to(device)
# 一行搞定，模型结构和权重一起从文件夹读取
```

---

## 🔗 关联笔记

- [[v2_04_两版核心对比]] → 综合对比
- [[07_predict]]（第一版预测）
