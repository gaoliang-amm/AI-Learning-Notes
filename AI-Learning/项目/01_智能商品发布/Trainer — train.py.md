```python
import time  
from dataclasses import dataclass  
from pathlib import Path  
  
import torch  
from huggingface_hub.utils import tqdm  
from sklearn.metrics import accuracy_score, f1_score  
from torch.utils.data import DataLoader  
from torch.utils.tensorboard import SummaryWriter  
from transformers import BertForSequenceClassification, AutoTokenizer, DataCollatorWithPadding  
  
from configuration import config  
from preprocess.dataset import get_dataset  
  
  
# class Training_Config:  
#     """存放训练过程中的超参数"""  
#  
#     def __init__(self, lr=5e-5, batch_size=16, epochs=10):  
#         self.lr = lr  
#         self.batch_size = batch_size  
#         self.epochs = epochs  
@dataclass  
class Training_Config:  
    epochs: int = 10  
    learning_rate: float = 5e-5  
    batch_size: int = 16  
    output_dir: str = './'  
    log_dir: str = './'  
    save_step: int = 20  
    early_stop_metric: str = 'loss'  
    patience: int = 3  
    amp_enabled: bool = True  
  
  
class Trainer:  
    def __init__(self,  
                 device,  
                 model,  
                 valid_dataset,  
                 compute_metrics,  
                 collate_fn,  
                 training_config=Training_Config(),  
                 train_dataset=None):  
        # 设备  
        self.device = device  
        # 模型  
        self.model = model.to(self.device)  
        # 数据  
        self.train_dataset = train_dataset  
        self.valid_dataset = valid_dataset  
        # 数据整理器  
        self.collate_fn = collate_fn  
        # 评估函数  
        self.compute_metrics = compute_metrics  
        # 优化器  
        self.optimizer = torch.optim.Adam(self.model.parameters(), lr=training_config.learning_rate)  
        # 训练参数  
        self.training_config = training_config  
        # 全局step  
        self.step = 0  
        # tensorboard  
        self.writer = SummaryWriter(str(Path(training_config.log_dir) / time.strftime("%Y-%m-%d-%H-%M-%S")))  
        # 全局最小loss  
        # self.best_loss = float("inf")  
        # 提前停止  
        self.early_stop_count = 0  
  
        # 最优分数  
        self.best_score = -float('inf')  
  
        # 定义amp  
        self.scaler = torch.amp.GradScaler("cuda", enabled=self.training_config.amp_enabled)  
  
    def _get_dataloader(self, dataset):  
        """根据数据集获取对应的dataloader"""  
        dataset.set_format("torch")  
        generator = torch.Generator()  
        generator.manual_seed(42)  
        return DataLoader(dataset=dataset,  
                          batch_size=self.training_config.batch_size,  
                          shuffle=True,  
                          collate_fn=self.collate_fn,  
                          generator=generator  
                          )  
  
    def train(self):  
        """训练"""  
        # 检查点加载，如果有检查点，没有就从头开始训练  
        self._load_checkpoint()  
        current_step = 0  
  
        dataloader = self._get_dataloader(self.train_dataset)  
        for epoch in range(1, 1 + self.training_config.epochs):  
            for batch in tqdm(dataloader, desc=f"EPOCHS:{epoch}"):  
  
                current_step += 1  
                if current_step <= self.step:  
                    continue  
  
                self.step += 1  
  
                loss = self._train_one_batch(batch)  
  
                if self.step % self.training_config.save_step == 0:  
  
                    # 模型保存  
                    self._save_checkpoint()  
  
                    # 记录loss  
                    self.writer.add_scalar("loss", loss, self.step)  
  
                    # 早停  
                    # 1. 验证评估-> 评估指标（验证集）  
                    metrics = self.evaluate()  
  
                    # Evaluation: loss:0.12 / accuracy:0.98 / f1:0.95  
                    metrics_str = "Evaluation: " + " | ".join(  
                        [f"{k}:{v:.4f}" for k, v in metrics.items()]  
                    )  
                    tqdm.write(metrics_str)  
  
                    # 2. 根据评估指标做早停判断  
                    if self._should_early_stop(metrics):  
                        return  
  
                    # 判断是否保存模型  
                    # if loss < self.best_loss:  
                    #     self.best_loss = loss                    #     # 保存模型  
                    #     print("保存模型！")  
                    #     tqdm.write(f"保存模型！")  
                    #     self.model.save_pretrained(self.training_config.output_dir)  
    def evaluate(self):  
        """验证评估"""  
        self.model.eval()  
  
        total_loss = 0  
        all_predictions = []  
        all_labels = []  
  
        dataloader = self._get_dataloader(self.valid_dataset)  
        for batch in tqdm(dataloader, desc="Evaluation"):  
            input = {k: v.to(self.device) for k, v in batch.items()}  
            with torch.no_grad():  
                # 前向传播  
                outputs = self.model(**input)  
            # 获取loss  
            loss = outputs.loss  
            total_loss += loss.item()  
  
            # 获取预测值  
            logits = outputs.logits  
            predictions = torch.argmax(logits, dim=-1)  
            all_predictions.extend(predictions.tolist())  
  
            # 获取真实值  
            labels = batch["labels"]  
            all_labels.extend(labels.tolist())  
  
        # 计算评估指标  
        ## 平均loss  
        avg_loss = total_loss / len(dataloader)  
  
        ## 准确率  
        metrics = self.compute_metrics(all_predictions, all_labels)  
  
        return {"loss": avg_loss, **metrics}  
  
    def _train_one_batch(self, batch):  
        """训练一个批次"""  
        self.model.train()  
        # 把数据放到设备  
        inputs = {k: v.to(self.device) for k, v in batch.items()}  
        with torch.autocast(device_type=self.device.type, dtype=torch.float16,  
                            enabled=self.training_config.amp_enabled):  
            # 前向传播  
            outputs = self.model(**inputs)  
        # 计算loss  
        loss = outputs.loss  
        # 梯度清零  
        self.optimizer.zero_grad()  
        # 反向传播  
        self.scaler.scale(loss).backward()  
        # 优化器更新参数  
        # self.optimizer.step()  
        self.scaler.step(self.optimizer)  
        # 更新系数  
        self.scaler.update()  
  
        return loss.item()  
  
    def _should_early_stop(self, metrics):  
        """  
        做早停判断  
        :param metrics:        :return:  
        """        # 获取当前指标  
        metric = metrics[self.training_config.early_stop_metric]  
  
        # 定义一个统一的分数，越大表示效果越好  
        score = -metric if self.training_config.early_stop_metric == 'loss' else metric  
        if score > self.best_score:  
            tqdm.write("保存模型...")  
            self.best_score = score  
            self.early_stop_count = 0  
            self.model.save_pretrained(self.training_config.output_dir)  
  
            return False  
        else:  
            self.early_stop_count += 1  
            if self.early_stop_count > self.training_config.patience:  
                tqdm.write("早停...")  
                return True  
            else:  
                return False  
  
    def _save_checkpoint(self):  
        """  
        保存检查点  
        1. 模型的权重  
        2. 优化器  
        3. 全局step  
        4. early_stop_count        5. best_score        6. scalar        :return:  
        """        checkpoint = {  
            "model": self.model.state_dict(),  
            "opt": self.optimizer.state_dict(),  
            "step": self.step,  
            "count": self.early_stop_count,  
            "score": self.best_score,  
            "scaler": self.scaler.state_dict()  
        }  
        file_name = str(config.MODELS_DIR / "checkpoint" / "checkpoint.pt")  
        torch.save(checkpoint, file_name)  
  
    def _load_checkpoint(self):  
        path = config.MODELS_DIR / "checkpoint" / "checkpoint.pt"  
        if path.exists():  
            tqdm.write("加载检查点！")  
            checkpoint = torch.load(path)  
            self.model.load_state_dict(checkpoint["model"])  
            self.optimizer.load_state_dict(checkpoint["opt"])  
            self.step = checkpoint["step"]  
            self.early_stop_count = checkpoint["count"]  
            self.best_score = checkpoint["score"]  
            self.scaler.load_state_dict(checkpoint["scaler"])  
        else:  
            tqdm.write("没有检查点！")  
  
  
def train():  
    # 设备  
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")  
    # 模型  
    with open(config.MODELS_DIR / "labels.txt", "r", encoding="utf-8") as f:  
        all_labels = f.read().split("\n")  
    id2label = {i: label for i, label in enumerate(all_labels)}  
    label2id = {label: i for i, label in enumerate(all_labels)}  
    model = BertForSequenceClassification.from_pretrained(config.PRETRAINED_DIR / 'bert-base-chinese',  
                                                          num_labels=len(all_labels),  
                                                          id2label=id2label,  
                                                          label2id=label2id)  
    # torch.save(model.state_dict(), config.MODELS_DIR / "model.pt")  
    # model.save_pretrained(config.MODELS_DIR)  
    # 分词器  
    tokenizer = AutoTokenizer.from_pretrained(config.PRETRAINED_DIR / 'bert-base-chinese')  
  
    # 数据  
    # dataloader = get_dataloader(tokenizer, "train")  
    train_dataset = get_dataset("train")  
    valid_dataset = get_dataset("valid")  
    collate_fn = DataCollatorWithPadding(tokenizer, padding=True, return_tensors="pt")  
  
  
    # 评估函数  
    def compute_metrics(predictions, labels) -> dict:  # {"accuracy": 0.9, "precision": 0.8, "recall": 0.9}  
        accuracy = accuracy_score(labels, predictions)  
        f1 = f1_score(labels, predictions, average="weighted")  
  
        return {"accuracy": accuracy, "f1": f1}  
  
  
    # 超参数  
    training_config = Training_Config(  
        output_dir=config.MODELS_DIR,  
        log_dir=config.LOGS_DIR  
    )  
  
    # 优化器  
    # optimizer = torch.optim.Adam(model.parameters(), lr=training_config.lr)  
  
    # 训练  
    trainer = Trainer(  
        device=device,  
        model=model,  
        train_dataset=train_dataset,  
        valid_dataset=valid_dataset,  
        compute_metrics=compute_metrics,  
        collate_fn=collate_fn,  
        training_config=training_config  
    )  
  
    trainer.train()  
  
if __name__ == '__main__':  
    train()
```