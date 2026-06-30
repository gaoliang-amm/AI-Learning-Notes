# 🏋️ train.py — 训练脚本

完整的模型训练流程，包含训练循环、损失计算、参数更新和模型保存。

---

## 训练整体流程

```
初始化 设备 / DataLoader / 模型 / 损失函数 / 优化器 / TensorBoard
        ↓
  for epoch in range(50):
        ↓
    train_one_epoch()  ←── 遍历所有 batch
        ↓
    记录损失到 TensorBoard
        ↓
    如果是最低损失 → 保存模型权重
```

---

## `train_one_epoch()` 逐步解析

```python
def train_one_epoch(model, dataloader, loss_fn, optimizer, device):
    model.train()       # 开启训练模式（启用 Dropout 等）
    total_loss = 0
    
    for batch in tqdm(dataloader, desc='Training'):
        # 取出标签，移到 GPU（float 因为 BCEWithLogitsLoss 需要浮点）
        targets = batch.pop('labels').to(device).float()
        # 剩余字段（input_ids 等）移到 GPU
        inputs = {k: v.to(device) for k, v in batch.items()}
        
        # ① 前向传播
        outputs = model(**inputs)           # shape: (batch_size,)
        
        # ② 计算损失
        loss = loss_fn(outputs, targets)
        
        # ③ 清零梯度（防止累积）
        optimizer.zero_grad()
        
        # ④ 反向传播（计算梯度）
        loss.backward()
        
        # ⑤ 更新参数
        optimizer.step()
        
        total_loss += loss.item()
    
    return total_loss / len(dataloader)     # 返回平均损失
```

### 训练的 5 步口诀

```
前向传播 → 计算损失 → 清零梯度 → 反向传播 → 更新参数
```

---

## `train()` 函数：初始化与主循环

```python
device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
# 有 GPU 用 GPU，没有用 CPU

train_loader = get_dataloader(train=True)

model = ReviewAnalyzeModel().to(device)

loss_fn = nn.BCEWithLogitsLoss()
# 二元交叉熵损失（适合二分类）
# 内部已包含 Sigmoid，所以模型输出不需要手动加 Sigmoid

optimizer = torch.optim.Adam(model.parameters(), lr=LEARNING_RATE)
# Adam 优化器，微调 BERT 常用

writer = SummaryWriter(log_dir=LOGS_DIR / time.strftime('%Y-%m-%d_%H-%M-%S'))
# TensorBoard 日志写入器，按时间戳建文件夹
```

### 保存最优模型

```python
min_loss = float('inf')  # 初始化为无穷大

for epoch in range(EPOCHS):
    this_loss = train_one_epoch(...)
    writer.add_scalar('Loss/train', this_loss, epoch + 1)
    
    if this_loss < min_loss:
        torch.save(model.state_dict(), MODELS_DIR / BEST_MODEL)
        min_loss = this_loss
        print("模型保存成功！")
```

> 💡 只保存权重（`state_dict()`），不保存整个模型对象，文件更小，加载也更灵活。

---

## 损失函数：BCEWithLogitsLoss

| 场景 | 推荐损失函数 |
|------|-------------|
| 二分类（本项目） | `BCEWithLogitsLoss` |
| 多分类 | `CrossEntropyLoss` |
| 回归 | `MSELoss` |

`BCEWithLogitsLoss(output, target)` 等价于先对 output 做 Sigmoid，再计算二元交叉熵：

```
loss = -[y·log(σ(x)) + (1-y)·log(1-σ(x))]
```

---

## 查看训练曲线

```bash
tensorboard --logdir logs/
```

然后浏览器打开 `http://localhost:6006`，查看 `Loss/train` 曲线。

---

## 🔗 关联笔记

- [[03_dataset]] → `get_dataloader()`
- [[04_model]] → `ReviewAnalyzeModel`
- [[01_config]] → `EPOCHS`, `BATCH_SIZE`, `LEARNING_RATE`
