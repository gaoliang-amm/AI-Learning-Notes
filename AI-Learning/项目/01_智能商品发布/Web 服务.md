### service.py
```python
class Title_Predict:  
    def __init__(self, predictor):  
        self.predictor = predictor  
  
    def predict(self, text: str | list):  
        return self.predictor.predict(text)
```
### schemas.py
```python
from pydantic import BaseModel  
  
class Title(BaseModel):  
    """商品标题类"""  
    name: str | list  
  
  
class Category(BaseModel):  
    """商品类目类"""  
    name: str | list
```
### app.py
```python
import torch  
import uvicorn  
from fastapi import FastAPI  
from transformers import BertForSequenceClassification, AutoTokenizer  
  
from configuration import config  
from runner.predict import Predictor  
from web.schemas import Title, Category  
from web.service import Title_Predict  
  
app = FastAPI(description="商品分类系统")  
  
# 创建predictor  
# 设备  
device = torch.device("cuda" if torch.cuda.is_available() else "cpu")  
  
# 模型  
model = BertForSequenceClassification.from_pretrained(config.MODELS_DIR)  
  
# tokenizer  
tokenizer = AutoTokenizer.from_pretrained(config.PRETRAINED_DIR / 'bert-base-chinese')  
  
# 预测器  
predictor = Predictor(device, model, tokenizer)  
  
# 创建一个标题分类类别  
title_predict = Title_Predict(predictor)  
  
@app.post("/product_classify")  
def get_classify(title: Title) -> Category:  
    res = predictor.predict(title.name)  
    return Category(name=res)  
  
def server():  
    uvicorn.run("web.app:app", host="0.0.0.0", port=8000)  
  
if __name__ == '__main__':  
    server()
```