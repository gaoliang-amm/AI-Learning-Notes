# 🧹 preprocess.py — 数据预处理

将原始 CSV 文件变成模型可以直接读取的格式，**只需运行一次**，结果保存到磁盘。

---

## 📊 原始数据格式

文件：`data/raw/online_shopping_10_cats.csv`

| 列名 | 含义 |
|------|------|
| `review` | 评论文本（中文） |
| `label` | 情感标签（0=负面，1=正面） |
| `cat` | 商品类别（不使用，会被删掉） |

---

## 🔄 处理流程（5 步）

```
原始 CSV
  ↓ 1. 读取
HuggingFace Dataset
  ↓ 2. 数据清洗（删列、去空值）
  ↓ 3. 划分训练集/测试集（8:2）
  ↓ 4. Tokenizer 编码
编码后的 Dataset
  ↓ 5. 保存到磁盘
data/processed/train + test
```

---

## 📝 代码逐行解析

### 第 1 步：读取 CSV

```python
dataset = load_dataset('csv', data_files=str(RAW_DATA_DIR / RAW_DATA_FILE))['train']
```

> HuggingFace `load_dataset` 加载 CSV，返回 `DatasetDict`，取 `'train'` 得到 `Dataset` 对象。

---

### 第 2 步：数据清洗

```python
dataset = dataset.remove_columns(['cat'])              # 删掉 cat 列，不需要
dataset = dataset.filter(lambda x: x['review'] is not None)  # 删掉空评论
```

---

### 第 3 步：划分数据集

```python
dataset = dataset.cast_column('label', ClassLabel(names=['neg', 'pos']))  # 把 label 声明为分类标签
dataset_dict = dataset.train_test_split(test_size=0.2, stratify_by_column='label')
```

> 💡 `stratify_by_column='label'`：**分层抽样**，保证训练集和测试集中正/负样本比例相同，避免数据不平衡。

结果：
- `dataset_dict['train']` → 训练集（80%）
- `dataset_dict['test']` → 测试集（20%）

---

### 第 4 步：Tokenizer 编码

```python
tokenizer = AutoTokenizer.from_pretrained(MODEL_NAME_OR_PATH)

def encode(batch):
    inputs = tokenizer(
        batch['review'],
        truncation=True,        # 超过 MAX_SEQ_LEN 截断
        padding='max_length',   # 不足 MAX_SEQ_LEN 补零
        max_length=MAX_SEQ_LEN  # = 128
    )
    inputs['labels'] = batch['label']
    return inputs

dataset_dict = dataset_dict.map(encode, batched=True, remove_columns=['review', 'label'])
```

编码后每条样本包含：

| 字段 | 含义 |
|------|------|
| `input_ids` | 文字变成数字 ID 序列（长度 128） |
| `attention_mask` | 哪些位置是真实文字（1），哪些是填充（0） |
| `token_type_ids` | 区分句子 A 和句子 B（单句任务全 0） |
| `labels` | 标签（0 或 1） |

---

### 第 5 步：保存

```python
dataset_dict.save_to_disk(PROCESSED_DATA_DIR)
```

保存为 Arrow 格式，后续 `load_from_disk` 快速加载，不用重复预处理。

---

## 🔗 关联笔记

- [[01_config]] → `MAX_SEQ_LEN`, `MODEL_NAME_OR_PATH`
- [[03_dataset]] → 加载这里保存的数据
- [[08_新手必读_核心概念]] → Tokenizer 是什么？
