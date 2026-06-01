```python
from datasets import load_dataset, ClassLabel, Value  
from transformers import AutoTokenizer  
  
from configuration import config  
  
  
def process():  
    """  
    数据预处理  
    :return:  
    """  
    # 1. 加载数据集  
    dataset_dict = load_dataset("csv", data_files={  
        "train": str(config.RAW_DATA_DIR / "train.txt"),  
        "test": str(config.RAW_DATA_DIR / "test.txt"),  
        "valid": str(config.RAW_DATA_DIR / "valid.txt")  
    }, delimiter="\t")  
  
    # 2. 数据清洗  
    dataset_dict = dataset_dict.filter(lambda x: x["label"] is not None or x['text_a'] is not None)  
  
    # 3. 处理label（分类标签）  
    ## 3.1 拿到所有不重复的标签，并排序  
    all_labels = sorted(set(dataset_dict["train"]["label"]))  
    ## 3.2 文字标签 → 数字标签 等价于3.3 cast_column，但是无法固定标签顺序  
    # dataset_dict = dataset_dict.class_encode_column("label")  
  
    ## 3.3 固定标签映射关系，保证训练 / 测试一致  
    dataset_dict = dataset_dict.cast_column("label", Value("string"))  
    dataset_dict = dataset_dict.cast_column("label", ClassLabel(names=all_labels))  
    # 保存all_labels  
    with open(config.MODELS_DIR / "labels.txt", "w", encoding="utf-8") as f:  
        f.write("\n".join(all_labels))  
  
    # 4. 处理商品标题（text_a）  
    ## 4.1 加载分词器  
    tokenizer = AutoTokenizer.from_pretrained(config.PRETRAINED_DIR / 'bert-base-chinese')  
  
    def tokenize(batch):  
        """  
        批量分词 + 保留标签  
        :param batch:        :return:  
        """        inputs = tokenizer(batch['text_a'], truncation=True, return_token_type_ids=False)  
        # 不设置max_length，则使用tokenizer.model_max_length模型最大长度做截断  
        inputs['labels'] = batch['label']  
        return inputs  
  
    dataset_dict = dataset_dict.map(tokenize, batched=True, remove_columns=['label', 'text_a'])  
  
    # 5. 保存数据集  
    dataset_dict.save_to_disk(config.PROCESSED_DATA_DIR)  
  
  
if __name__ == '__main__':  
    process()  
  
"""  
map 是 Hugging Face 数据集处理中最主流、最标准、性能最好的一种方案  
自动批处理  
自动多进程  
自动缓存  
Arrow 格式高效  
不会一次性爆内存  
"""
```