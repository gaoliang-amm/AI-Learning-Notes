# 📦 dataset.py — 数据集加载器

封装了从磁盘加载预处理数据、并转成 PyTorch `DataLoader` 的逻辑。

---

## 核心函数：`get_dataloader(train=True)`

```python
def get_dataloader(train=True):
    # 1. 确定加载哪个子集
    data_path = PROCESSED_DATA_DIR / ('train' if train else 'test')
    
    # 2. 从磁盘加载 HuggingFace Dataset
    dataset = load_from_disk(data_path)
    
    # 3. 转成 PyTorch Tensor 格式
    dataset.set_format(type='torch')
    
    # 4. 封装成 DataLoader
    dataloader = DataLoader(dataset, batch_size=BATCH_SIZE, shuffle=train)
    
    return dataloader
```

---

## 参数说明

| 参数 | 含义 |
|------|------|
| `train=True` | 加载训练集；`False` 加载测试集 |
| `batch_size=BATCH_SIZE` | 每次从数据集取 16 条 |
| `shuffle=train` | 训练时打乱顺序，测试时不打乱 |

---

## DataLoader 的作用

> 💡 把整个数据集切成一个个**小批次（batch）**，训练时循环取用。

```
Dataset（全量数据）
    ↓  DataLoader（按 batch_size=16 切分）
[batch_1: 16条] → [batch_2: 16条] → ... → [batch_N: 16条]
```

每个 `batch` 是一个字典：
```python
{
  'input_ids':      Tensor(shape=[16, 128]),
  'attention_mask': Tensor(shape=[16, 128]),
  'token_type_ids': Tensor(shape=[16, 128]),
  'labels':         Tensor(shape=[16])
}
```

---

## 测试代码（`if __name__ == '__main__'`）

```python
train_loader = get_dataloader(train=True)
test_loader  = get_dataloader(train=False)

print(len(train_loader), len(test_loader))  # 打印批次数量

for batch in train_loader:
    for k, v in batch.items():
        print(k, "->", v.shape)
    break  # 只看第一批
```

---

## 🔗 关联笔记

- [[02_preprocess]] → 生成 `processed/` 数据
- [[05_train]] → 调用此函数获取训练数据
- [[06_evaluate]] → 调用此函数获取测试数据
