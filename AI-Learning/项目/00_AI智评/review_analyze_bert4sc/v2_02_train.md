# 🏋️ v2 train.py — 训练（自动损失 + 新保存方式）

本版训练脚本有两处关键变化：损失函数不用手动定义，模型保存格式不同。

---

## 变化一：损失由模型自动计算

第一版需要手动定义损失函数：

```python
# 第一版
loss_fn = nn.BCEWithLogitsLoss()
...
outputs = model(**inputs)        # 输出标量
loss = loss_fn(outputs, targets) # 手动传入损失函数
```

本版只需把 `labels` 一起传给模型，损失自动返回：

```python
# 本版
# loss_fn = nn.BCEWithLogitsLoss()  ← 注释掉了，不需要了

inputs = {k: v.to(device) for k, v in batch.items()}
# batch 里已包含 labels，一起传入
outputs = model(**inputs)

loss = outputs.loss   # ← 直接从输出对象取损失！
```

> 💡 `AutoModelForSequenceClassification` 在检测到输入包含 `labels` 时，**自动在内部计算交叉熵损失**并放到 `outputs.loss` 里。

---

## `train_one_epoch` 完整代码

```python
def train_one_epoch(model, dataloader, optimizer, device):
    model.train()
    total_loss = 0
    for batch in tqdm(dataloader, desc='Training'):
        inputs = {k: v.to(device) for k, v in batch.items()}  # 包含 labels

        outputs = model(**inputs)    # 前向传播
        loss = outputs.loss          # 自动损失

        optimizer.zero_grad()        # 清零梯度
        loss.backward()              # 反向传播
        optimizer.step()             # 更新参数

        total_loss += loss.item()
    return total_loss / len(dataloader)
```

注意：第一版需要单独 `batch.pop('labels')` 再传，本版直接把整个 batch 传入即可，更简洁。

---

## 变化二：模型保存方式

第一版（只保存权重文件）：

```python
torch.save(model.state_dict(), MODELS_DIR / BEST_MODEL)
# 保存成：models/best_model.pt
```

本版（保存整个模型文件夹）：

```python
model.save_pretrained(MODELS_DIR)
# 保存成：models/ 文件夹，包含 config.json + 权重文件
```

| 对比 | `torch.save` | `save_pretrained` |
|------|-------------|------------------|
| 保存内容 | 只有权重 | 权重 + 模型配置 |
| 加载方式 | 需要先建模型对象，再 `load_state_dict` | 直接 `from_pretrained(路径)` |
| 可移植性 | 低（需要有同样的模型代码） | 高（自描述，HuggingFace 标准格式） |

---

## 变化三：模型初始化

第一版：
```python
model = ReviewAnalyzeModel().to(device)   # 自定义类
```

本版：
```python
model = AutoModelForSequenceClassification.from_pretrained(MODEL_NAME_OR_PATH).to(device)
```

> `AutoModelForSequenceClassification` 会在 BERT 上自动加一个分类头（默认 2 分类），不需要我们手写 `model.py`。

---

## 🔗 关联笔记

- [[v2_04_两版核心对比]] → 更深入的对比
- [[05_train]]（第一版训练）
