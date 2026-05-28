# ⚙️ config.py — 全局配置

所有路径和超参数都集中写在这一个文件里，其他模块直接 `from config import *` 引用，方便统一管理。

---

## 📁 目录路径配置

```python
from pathlib import Path

ROOT_DIR = Path(__file__).parent.parent   # 项目根目录（src 的上一级）

DATA_DIR          = ROOT_DIR / 'data'
RAW_DATA_DIR      = DATA_DIR / 'raw'       # 原始 CSV 放这里
PROCESSED_DATA_DIR= DATA_DIR / 'processed' # 预处理后的数据集

LOGS_DIR   = ROOT_DIR / 'logs'    # TensorBoard 日志
MODELS_DIR = ROOT_DIR / 'models'  # 保存训练好的模型
```

> 💡 `Path(__file__).parent.parent`：`__file__` 是当前脚本路径，`.parent` 是上一级目录，`.parent.parent` 是上两级 → 即项目根目录。

---

## 📄 文件名配置

```python
RAW_DATA_FILE = 'online_shopping_10_cats.csv'  # 原始数据文件名
BEST_MODEL    = 'best_model.pt'                # 最优模型保存文件名
```

---

## 🤗 预训练模型配置

```python
MODEL_NAME_OR_PATH = '../../mybert'
```

> 💡 这里使用**本地 BERT 模型**（相对路径指向 `mybert` 文件夹）。也可以换成 HuggingFace Hub 上的模型名，如 `'bert-base-chinese'`，会自动联网下载。

---

## 🎛️ 训练超参数

| 参数 | 值 | 含义 |
|------|----|------|
| `EPOCHS` | `50` | 训练轮次（整个数据集过 50 遍） |
| `BATCH_SIZE` | `16` | 每批次处理 16 条样本 |
| `MAX_SEQ_LEN` | `128` | 文本最大 Token 长度（超出截断） |
| `LEARNING_RATE` | `1e-5` | 学习率（0.00001，BERT 微调典型值） |

> ⚠️ EPOCHS=50 对于小数据集可能导致过拟合，实际调参时可适当减小。

---

## 🔗 关联笔记

- [[00_项目总览]]
- [[08_新手必读_核心概念]] → 超参数是什么？
