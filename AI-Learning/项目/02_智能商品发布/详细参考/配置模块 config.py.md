```python
from pathlib import Path  
  
ROOT_DIR = Path(__file__).parent.parent.parent  
  
# 源数据目录  
RAW_DATA_DIR = ROOT_DIR / "data" / "raw"  
  
# 处理后的数据目录  
PROCESSED_DATA_DIR = ROOT_DIR / "data" / "processed"  
  
# 日志目录  
LOGS_DIR = ROOT_DIR / "logs"  
  
# 模型目录  
MODELS_DIR = ROOT_DIR / "models"  
  
# 保存预训练模型的目录  
PRETRAINED_DIR = ROOT_DIR / "pretrained"
```