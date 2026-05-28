# 📊 evaluate.py — 模型评估

用测试集评估训练好的模型准确率。

---

## 核心函数 `evaluate()`

```python
def evaluate(model, dataloader, device):
    model.eval()        # 关闭 Dropout，固定 BatchNorm
    correct_cnt = 0
    total_cnt = 0
    
    with torch.no_grad():   # 不计算梯度（节省内存和计算）
        for batch in tqdm(dataloader, desc='评估模型'):
            targets = batch.pop('labels').tolist()
            inputs = {k: v.to(device) for k, v in batch.items()}
            
            # 得到预测概率（Sigmoid 后的值）
            batch_probs = predict_batch(model, inputs, device)
            
            for prob, target in zip(batch_probs, targets):
                total_cnt += 1
                pred_label = 1 if prob > 0.5 else 0   # 概率>0.5 → 正面
                if pred_label == target:
                    correct_cnt += 1
    
    return correct_cnt / total_cnt   # 准确率
```

---

## `run_evaluate()` 入口

```python
def run_evaluate():
    device = torch.device('cuda' if torch.cuda.is_available() else 'cpu')
    model = ReviewAnalyzeModel().to(device)
    model.load_state_dict(torch.load(MODELS_DIR / BEST_MODEL))   # 加载权重
    
    train_loader = get_dataloader(train=False)   # ← 注意：加载的是测试集
    acc = evaluate(model, train_loader, device)
    print(f"准确率: {acc:.4f}")
```

> ⚠️ 代码中变量名写的是 `train_loader`，但 `train=False` 其实加载的是**测试集**，变量命名有误，不影响逻辑。

---

## 准确率的含义

```
准确率 = 预测正确的样本数 / 总样本数
```

BERT 在中文情感分类任务上，通常能达到 **90%+** 的准确率。

---

## 🔗 关联笔记

- [[04_model]] → `ReviewAnalyzeModel`
- [[07_predict]] → `predict_batch()` 函数
