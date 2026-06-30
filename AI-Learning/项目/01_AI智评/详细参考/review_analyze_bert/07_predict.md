# 🔮 predict.py — 推理预测

提供三个层次的预测接口：批量预测、单条预测、交互式命令行应用。

---

## 三个函数概览

| 函数 | 输入 | 输出 | 用途 |
|------|------|------|------|
| `predict_batch()` | 已编码的 batch 数据 | 概率列表 | 底层批量推理 |
| `predict()` | 原始文本字符串 | 单个概率值 | 单条预测 |
| `run_app()` | 键盘输入 | 打印结果 | 交互式应用 |

---

## `predict_batch()` — 批量预测

```python
def predict_batch(model, inputs, device):
    model.eval()
    with torch.no_grad():
        inputs = {k: v.to(device) for k, v in inputs.items()}
        outputs = model(**inputs)        # 模型原始输出（未经 Sigmoid）
    
    batch_probs = torch.sigmoid(outputs) # 转为概率 [0, 1]
    return batch_probs.tolist()          # 转为 Python 列表返回
```

> 💡 这里手动加 `sigmoid`，因为训练时用的 `BCEWithLogitsLoss` 已内置 sigmoid；**推理时要自己加**。

---

## `predict()` — 单条文本预测

```python
def predict(text, model, tokenizer, device):
    # 1. 文本 → 编码
    inputs = tokenizer(
        text,
        truncation=True,
        padding='max_length',
        max_length=MAX_SEQ_LEN,
        return_tensors='pt'    # 返回 PyTorch Tensor
    )
    # 2. 推理
    result = predict_batch(model, inputs, device)
    # 3. 返回正向概率
    return result[0]
```

---

## `run_app()` — 交互式命令行应用

```python
def run_app():
    device = ...
    tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME_OR_PATH)
    model = ReviewAnalyzeModel().to(device)
    model.load_state_dict(torch.load(MODELS_DIR / BEST_MODEL))
    
    print("欢迎使用情感分析小程序！输入q或者quit退出...")
    while True:
        text = input("评论> ")
        if text in ('q', 'quit'):
            break
        if text.strip() == '':
            print("请输入有效内容！")
            continue
        
        result = predict(text, model, tokenizer, device)
        
        if result > 0.5:
            print("预测结果：", "正面", result)
        else:
            print("预测结果：", "负面", 1 - result)
```

### 运行示例

```
欢迎使用情感分析小程序！输入q或者quit退出...
评论> 东西很好，下次还买
预测结果： 正面 0.9823
评论> 质量太差了，根本不值这个价
预测结果： 负面 0.9541
评论> q
欢迎下次再来！
```

---

## 输出解读

| 输出 | 含义 |
|------|------|
| `正面 0.98` | 98% 概率是正面评论 |
| `负面 0.95` | 负面概率 = `1 - 正向概率`，此处正向概率约 0.05 |

---

## 🔗 关联笔记

- [[04_model]] → `ReviewAnalyzeModel`
- [[06_evaluate]] → 也调用了 `predict_batch()`
