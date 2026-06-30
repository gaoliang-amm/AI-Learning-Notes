```python
from datasets import load_from_disk  
from transformers import DataCollatorWithPadding, AutoTokenizer  
  
from configuration import config  
from torch.utils.data import DataLoader  
  
  
def get_dataset(dtype="train"):  
    return load_from_disk(config.PROCESSED_DATA_DIR / dtype)  
  
  
def get_dataloader(tokenizer, dtype="train"):  
    dataset = load_from_disk(config.PROCESSED_DATA_DIR / dtype)  
    dataset.set_format("torch")  
    return DataLoader(dataset,  
                      batch_size=16,  
                      shuffle=True,  
                      drop_last=True,  
                      collate_fn=DataCollatorWithPadding(tokenizer, padding=True, return_tensors="pt"))  
  
  
if __name__ == '__main__':  
    tokenizer = AutoTokenizer.from_pretrained(config.PRETRAINED_DIR / 'bert-base-chinese')  
    dataloader = get_dataloader(tokenizer, "train")  
    for batch in dataloader:  
        for k, v in batch.items():  
            print(k, v)  
        break
```