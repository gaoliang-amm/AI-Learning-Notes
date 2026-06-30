```python
import torch  
from transformers import BertForSequenceClassification, AutoTokenizer  
  
from configuration import config  
  
  
class Predictor:  
    """  
    实现一个predictor类，完成通用的预测逻辑  
    """  
    def __init__(self, device, model, tokenizer):  
        self.device = device  
        self.model = model.to(self.device)  
        self.tokenizer = tokenizer  
  
    def predict(self, text: str | list):  
        """  
        预测  前向传播  
        :param text:        :return:  
        """        is_str = isinstance(text, str)  
  
        if is_str:  
            text = [text]  
  
        # 分词处理  
        inputs = self.tokenizer(text, padding=True, truncation=True, return_tensors="pt")  
  
        # 放到设备  
        inputs = {k: v.to(self.device) for k, v in inputs.items()}  
  
        # 前向传播  
        outputs = self.model(**inputs)  
  
        # 获取预测值  
        logits = outputs.logits  
        predictions = torch.argmax(logits, dim=-1)  
  
        # 获取预测结果  
        result = [self.model.config.id2label[prediction.item()] for prediction in predictions]  
  
        if is_str:  
            return result[0]  
        else:  
            return result  
  
  
def predict():  
    # 设备  
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")  
  
    # 模型  
    model = BertForSequenceClassification.from_pretrained(config.MODELS_DIR)  
  
    # tokenizer  
    tokenizer = AutoTokenizer.from_pretrained(config.PRETRAINED_DIR / 'bert-base-chinese')  
  
    # 预测器  
    predictor = Predictor(device, model, tokenizer)  
  
    # 预测  
    res1 = predictor.predict("瓦伦丁Wurenbacher小麦西柚啤酒500ml*12听整箱装德国原装进口果啤")  
    res2 = predictor.predict(["潘婷丝质顺滑洗发露750ml", "脉动水蜜桃味1L"])  
    print(res1)  
    print(res2)
```