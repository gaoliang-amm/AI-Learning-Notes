# Transformer（Attention Is All You Need）面试讲解稿

## 一、Transformer 是什么

Transformer 是 Google 在 2017 年论文《Attention Is All You Need》中提出的深度学习模型。

它最大的特点是：

- 完全抛弃了 RNN 和 CNN
    
- 核心机制是 Self-Attention（自注意力机制）
    
- 能够并行计算
    
- 更擅长处理长距离依赖
    

现在的大模型：

- OpenAI 的 GPT 系列
    
- Google 的 BERT、Gemini
    
- Meta 的 Llama
    

本质上都建立在 Transformer 架构之上。

---

# 二、Transformer 整体结构

Transformer 整体由两部分组成：

```text
Encoder（编码器）
Decoder（解码器）
```

原论文中：

- Encoder：6层
    
- Decoder：6层
    

每层结构完全相同。

整体流程如下：

```text
输入文本
   ↓
Embedding
   ↓
Positional Encoding
   ↓
N层 Encoder
   ↓
Encoder 输出
   ↓
N层 Decoder
   ↓
Linear
   ↓
Softmax
   ↓
预测下一个词
```

Transformer 本质上是在做：

```text
Token 与 Token 之间的关系建模
```

---

# 三、输入部分

Transformer 输入主要包含两部分：

## 1. Embedding（词向量）

首先把每个单词转换成向量。

例如：

```text
I love AI
```

会变成：

```text
[512维向量]
[512维向量]
[512维向量]
```

Embedding 的作用是：

```text
把离散文本映射到连续向量空间
```

这样模型才能计算。

---

## 2. Positional Encoding（位置编码）

因为 Transformer 没有 RNN 的时间顺序结构。

所以模型本身不知道：

```text
谁在前
谁在后
```

因此需要加入位置编码。

原论文使用的是：

```text
sin 和 cos 函数
```

位置编码公式：

PE(pos,2i)=\sin\left(\frac{pos}{10000^{2i/d_{model}}}\right),\quad PE(pos,2i+1)=\cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)

位置编码的作用是：

- 给模型提供顺序信息
    
- 让模型知道 token 的相对位置
    

后来很多模型改成了：

```text
Learned Positional Embedding
RoPE
ALiBi
```

例如 GPT、Llama 中常见的 RoPE。

---

# 四、Encoder 结构（重点）

每一个 Encoder Layer 主要包含两部分：

```text
1. Multi-Head Self Attention
2. Feed Forward Network
```

并且每一层后面都会加入：

```text
Residual Connection（残差连接）
LayerNorm（层归一化）
```

整体结构：

```text
输入
 ↓
Multi-Head Attention
 ↓
Add & LayerNorm
 ↓
Feed Forward
 ↓
Add & LayerNorm
```

---

# 五、Self-Attention（核心）

Transformer 最核心的部分就是：

```text
Self-Attention
```

它解决的问题是：

```text
当前 token 应该关注哪些 token
```

例如：

```text
The animal didn't cross the street because it was tired
```

模型需要知道：

```text
it 指代 animal
```

Self-Attention 可以直接建立这种长距离关系。

---

# 六、Q、K、V 机制

输入向量后：

Transformer 会生成三组向量：

```text
Q（Query）
K（Key）
V（Value）
```

它们分别通过矩阵变换得到：

```text
Q = XWQ
K = XWK
V = XWV
```

其中：

- Query：我要找什么
    
- Key：我有什么特征
    
- Value：真正的信息内容
    

---

# 七、Attention 计算过程（重点）

Attention 公式：

Attention(Q,K,V)=\operatorname{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V

---

## 第一步：Q × K^T

```text
计算 token 之间的相关性
```

谁和谁更相似。

---

## 第二步：除以 sqrt(dk)

作用：

```text
防止数值过大
```

否则：

```text
softmax 梯度容易消失
```

---

## 第三步：softmax

得到注意力权重。

例如：

```text
当前词对其它词的关注程度
```

---

## 第四步：乘 V

对信息进行加权求和。

最终得到：

```text
融合上下文后的表示
```

---

# 八、Multi-Head Attention

Transformer 不只做一次 Attention。

而是：

```text
并行做多次 Attention
```

这就是：

```text
Multi-Head Attention
```

例如：

- Head1 学语法关系
    
- Head2 学语义关系
    
- Head3 学主谓关系
    

最后把多个 head 的结果拼接。

作用是：

```text
增强模型表达能力
```

---

# 九、Feed Forward Network

Attention 后：

每个 token 还会经过一个前馈神经网络。

结构通常是：

```text
Linear
↓
ReLU / GELU
↓
Linear
```

作用：

- 增强非线性能力
    
- 做特征变换
    

---

# 十、Residual Connection（残差连接）

公式：

```text
x + F(x)
```

作用：

- 防止梯度消失
    
- 方便训练深层网络
    

这个思想来自：

```text
ResNet
```

---

# 十一、LayerNorm（层归一化）

作用：

- 稳定训练
    
- 加快收敛
    

相比 BatchNorm：

```text
LayerNorm 不依赖 batch
```

因此更适合 NLP。

---

# 十二、Decoder 结构

Decoder 比 Encoder 多了一个模块：

```text
Cross Attention
```

Decoder 结构：

```text
Masked Self Attention
↓
Cross Attention
↓
Feed Forward
```

---

# 十三、Masked Self-Attention

Decoder 在生成文本时：

```text
不能看到未来信息
```

例如：

```text
I love ?
```

模型不能提前看到后面的词。

因此会加入：

```text
Mask
```

通常是：

```text
上三角矩阵 Mask
```

这样模型只能看到前面的 token。

---

# 十四、Cross Attention

Cross Attention 用于：

```text
Decoder 读取 Encoder 输出
```

其中：

```text
Q 来自 Decoder
K 和 V 来自 Encoder
```

作用：

```text
生成时参考输入信息
```

这是机器翻译的关键。

---

# 十五、输出层

Decoder 输出后：

```text
Linear
↓
Softmax
```

最终得到：

```text
下一个 token 的概率分布
```

例如：

```text
P(world)=0.8
```

---

# 十六、Transformer 的优势

## 1. 并行计算

RNN：

```text
必须按时间顺序计算
```

Transformer：

```text
整个序列同时计算
```

GPU 利用率更高。

---

## 2. 长距离依赖能力强

RNN：

```text
距离远容易遗忘
```

Transformer：

```text
任意 token 可以直接建立关系
```

---

## 3. 更容易扩展

Transformer 非常适合：

```text
大规模参数训练
```

因此成为：

```text
大语言模型基础架构
```

---

# 十七、Transformer 的缺点

Self-Attention 的时间复杂度：

O(n^2)

因为：

```text
每个 token 都要和其它 token 计算关系
```

所以：

```text
长文本显存消耗很大
```

后来出现了：

- FlashAttention
    
- Sparse Attention
    
- Linear Attention
    

等优化方案。

---

# 十八、GPT 与 BERT

## GPT

特点：

```text
只使用 Decoder
```

属于：

```text
自回归生成模型
```

适合：

- 文本生成
    
- 对话
    
- 写作
    

---

## BERT

特点：

```text
只使用 Encoder
```

属于：

```text
双向理解模型
```

适合：

- 分类
    
- 检索
    
- 信息抽取
    

---

# 十九、总结（面试结尾）

Transformer 的本质是：

```text
通过 Attention 建立 token 之间的全局关系
```

它把传统的：

```text
序列建模
```

转化成了：

```text
关系建模
```

这也是 Transformer 能成为现代大模型基础架构的核心原因。