### evaluate.py
```python
import torch  
from sklearn.metrics import accuracy_score, f1_score, precision_score  
from transformers import BertForSequenceClassification, DataCollatorWithPadding, AutoTokenizer  
  
from configuration import config  
from preprocess.dataset import get_dataset  
from runner.train import Trainer  
  
  
def evaluate():  
    """测试集评估"""  
    # 设备  
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")  
  
    # 模型  
    model = BertForSequenceClassification.from_pretrained(config.MODELS_DIR)  
  
    # 数据  
    test_dataset = get_dataset("test")  
  
    # tokenizer  
    tokenizer = AutoTokenizer.from_pretrained(config.PRETRAINED_DIR / 'bert-base-chinese')  
  
    # collate_fn  
    collate_fn = DataCollatorWithPadding(tokenizer, padding=True, return_tensors="pt")  
  
    # compute_metrics  
    def compute_metrics(predictions, labels) -> dict:  # {"accuracy": 0.9, "precision": 0.8, "recall": 0.9}  
        accuracy = accuracy_score(labels, predictions)  
        precision = precision_score(labels, predictions, average="weighted")  
        f1 = f1_score(labels, predictions, average="weighted")  
        return {"accuracy": accuracy, "f1": f1, "precision": precision}  
  
    trainer = Trainer(  
        device=device,  
        model=model,  
        valid_dataset=test_dataset,  
        compute_metrics=compute_metrics,  
        collate_fn=collate_fn  
    )  
  
    metrics = trainer.evaluate()  
    print(metrics)  
  
if __name__ == '__main__':  
    evaluate()
```