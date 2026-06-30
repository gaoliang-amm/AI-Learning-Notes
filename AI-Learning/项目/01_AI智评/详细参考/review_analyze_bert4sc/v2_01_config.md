# ⚙️ v2 config.py — 配置变化

本版配置与第一版几乎相同，只有一处改动。

---

## 变化：去掉了 `BEST_MODEL`

第一版：
```python
BEST_MODEL = 'best_model.pt'    # ← 有这一行
```

本版：
```python
# BEST_MODEL = 'best_model.pt'  # ← 注释掉了，不再需要
```

**为什么不需要了？**

第一版手动保存权重文件（`.pt`），需要一个文件名。本版用 `save_pretrained()` 保存整个模型文件夹，路径直接用 `MODELS_DIR`，不需要单独的文件名。

---

## 完整配置（未变化部分）

```python
ROOT_DIR = Path(__file__).parent.parent

DATA_DIR           = ROOT_DIR / 'data'
RAW_DATA_DIR       = DATA_DIR / 'raw'
PROCESSED_DATA_DIR = DATA_DIR / 'processed'
LOGS_DIR           = ROOT_DIR / 'logs'
MODELS_DIR         = ROOT_DIR / 'models'   # ← 本版保存的是整个文件夹

RAW_DATA_FILE      = 'online_shopping_10_cats.csv'
MODEL_NAME_OR_PATH = '../../mybert'

EPOCHS        = 50
BATCH_SIZE    = 16
MAX_SEQ_LEN   = 128
LEARNING_RATE = 1e-5
```

---

## `save_pretrained` 会保存什么？

```
models/
├── config.json          # 模型结构配置
├── pytorch_model.bin    # 模型权重（或 model.safetensors）
├── tokenizer_config.json
└── vocab.txt
```

这是标准的 HuggingFace 模型格式，可以直接用 `from_pretrained(路径)` 加载，也可以上传到 HuggingFace Hub。

---

## 🔗 关联笔记

- [[v2_00_项目总览]]
- [[v2_02_train]] → 使用 `save_pretrained`
- [[01_config]]（第一版对比）
