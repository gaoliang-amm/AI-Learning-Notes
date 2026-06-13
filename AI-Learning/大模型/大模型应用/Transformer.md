
**标签：** #Transformer #Attention #LLM #NLP #深度学习 #面试八股 #大模型基础
**关联：** [[Attention机制]] [[BERT]] [[GPT]] [[T5]] [[大模型微调]] [[HuggingFace]]

---

# 一、为什么需要 Transformer？

## 1. RNN 时代的问题

在 Transformer 出现之前，NLP 领域主要使用 RNN、LSTM、GRU。虽然能处理序列数据，但存在两个核心问题：

### （1）无法并行计算

```text
I love deep learning
```

RNN 必须按顺序执行：

```text
I → love → deep → learning
```

后面的计算依赖前面的结果，GPU 并行能力无法充分利用。

### （2）长距离依赖问题

```text
The animal didn't cross the street because it was too tired.
```

模型需要知道 `it → animal`，但两者距离很远。RNN 随着序列变长会出现梯度消失和记忆衰减，难以捕捉远距离语义关系。

## 2. Transformer 的诞生

2017 年 Google 发表论文 **《Attention Is All You Need》**，提出完全放弃 RNN，完全依赖 Attention。

核心思想：

```text
让每个单词直接看到所有单词
```

而不是像 RNN 那样逐个传递信息。

---

# 二、Transformer 整体结构

Transformer 由两部分组成：**Encoder（编码器）+ Decoder（解码器）**，原论文各 6 层。

```text
输入文本
   ↓
Embedding + Positional Encoding
   ↓
┌─────────────────┐
│ Encoder × 6     │  ← 理解输入内容
│  Multi-Head Attn │
│  Add & LayerNorm │
│  Feed Forward    │
│  Add & LayerNorm │
└─────────────────┘
   ↓
┌─────────────────┐
│ Decoder × 6     │  ← 生成目标内容
│  Masked Attn     │
│  Cross Attn      │
│  Feed Forward    │
│  Add & LayerNorm │
└─────────────────┘
   ↓
Linear → Softmax → 预测下一个 token
```

> [!note] 面试一句话
> Transformer 的本质是：**通过 Attention 建立 token 之间的全局关系**，把传统的"序列建模"转化成了"关系建模"。

---

# 三、输入部分

## 1. Embedding（词向量）

把每个 token 转换成向量，将离散文本映射到连续向量空间。

```text
I love AI
→ [512维向量] [512维向量] [512维向量]
```

## 2. Positional Encoding（位置编码）

Transformer 没有 RNN 的时间顺序结构，模型本身不知道谁在前谁在后，因此需要加入位置编码。

原论文使用 sin/cos 函数：

$$PE_{(pos,2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right), \quad PE_{(pos,2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

> [!tip] 为什么用 sin/cos？
> 因为 sin(a+b) 可以用 sin(a) 和 cos(b) 线性表示，模型可以通过线性变换学到**相对位置关系**。

后来的模型改用了其他方案：

| 方案 | 使用模型 |
|---|---|
| Learned Positional Embedding | BERT、GPT-1/2 |
| RoPE（旋转位置编码） | Llama、Qwen、GPT-NeoX |
| ALiBi | BLOOM |

---

# 四、Encoder 结构

每个 Encoder Layer 包含两个核心子层：

```text
输入
 ↓
Multi-Head Self Attention  ← token 之间的信息交互
 ↓
Add & LayerNorm
 ↓
Feed Forward Network       ← 每个 token 内部的非线性变换
 ↓
Add & LayerNorm
```

---

# 五、Self-Attention（核心）

## 1. 解决什么问题？

Self-Attention 解决的问题是：**当前 token 应该关注哪些 token？**

```text
The animal didn't cross the street because it was tired
```

处理 `it` 时，模型需要知道 `it` 指代 `animal`。Self-Attention 可以直接建立这种长距离关系，不需要像 RNN 一样逐个传递。

## 2. Q、K、V 机制

输入向量 X 通过三个矩阵变换得到三组向量：

```text
Q = X · W_Q    （Query：我要找什么）
K = X · W_K    （Key：  我有什么特征）
V = X · W_V    （Value：真正的信息内容）
```

## 3. Attention 计算过程

$$Attention(Q, K, V) = softmax\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

| 步骤 | 操作 | 作用 |
|---|---|---|
| 第一步 | $Q \times K^T$ | 计算 token 之间的相关性分数 |
| 第二步 | $\div \sqrt{d_k}$ | 缩放，防止数值过大导致 softmax 梯度消失 |
| 第三步 | Softmax | 归一化为注意力权重（0~1） |
| 第四步 | $\times V$ | 对信息加权求和，得到融合上下文后的表示 |

### 为什么要除以 $\sqrt{d_k}$？

当 $d_k$ 较大时，$QK^T$ 的方差会随维度增大，导致 softmax 输出趋近 one-hot（梯度接近 0）。除以 $\sqrt{d_k}$ 使方差稳定在 1 附近，softmax 输出更平滑。

### 举例理解

```text
The animal crossed the street because it was tired.
```

计算 `it` 的注意力时，`animal` 的权重最高 → 模型理解 `it = animal`。

> [!note] 面试一句话
> Self-Attention 本质上是：通过计算序列内部任意两个位置之间的相关性，实现全局信息交互。

---

# 六、Multi-Head Attention

## 为什么需要多个 Head？

一个 Attention 只能学习一种关系。Transformer 并行做多次 Attention（原论文 8 个 Head），不同 Head 关注不同关系：

```text
Head1 → 主谓关系    Head2 → 指代关系
Head3 → 语义关系    Head4 → 时态关系
```

计算流程：

```text
Q, K, V
   ↓ 拆分为 h 个 Head
Head1: Attn(Q1, K1, V1)
Head2: Attn(Q2, K2, V2)
...
Headh: Attn(Qh, Kh, Vh)
   ↓ Concat 拼接
   ↓ Linear 线性变换
输出
```

> [!note] 面试要点
> 每个头的维度 $d_k = d_{model} / h$，所以增加头数不会增加总计算量，只是把信息分散到不同子空间学习。

---

# 七、Feed Forward Network（FFN）

每个 token 经过 Attention 后，还会经过一个前馈网络：

```text
Linear → ReLU/GELU → Linear
```

| 模块 | 负责什么 |
|---|---|
| Attention | Token **之间**的信息交换（全局交互） |
| FFN | 每个 Token **内部**的非线性变换（特征提取） |

> [!note] 面试回答
> Attention 负责 Token 之间的信息交换，FFN 负责每个 Token 内部的非线性变换。两者配合完成完整的特征提取。

---

# 八、Residual Connection 与 LayerNorm

## Residual Connection（残差连接）

$$output = x + F(x)$$

作用：缓解梯度消失，让深层网络可以训练。思想来自 ResNet。

## LayerNorm（层归一化）

作用：稳定训练、加速收敛。

相比 BatchNorm：LayerNorm 不依赖 batch size，每个样本独立归一化，更适合 NLP。

> [!tip] 面试要点
> Transformer 能堆几十甚至上百层，核心原因就是 **Residual + LayerNorm** 的组合。

---

# 九、Decoder 结构

Decoder 比 Encoder 多一个子层（Cross Attention）：

```text
Masked Self Attention     ← 只能看左边（防止看到未来）
       ↓
Add & LayerNorm
       ↓
Cross Attention            ← Q 来自 Decoder，K/V 来自 Encoder
       ↓
Add & LayerNorm
       ↓
Feed Forward
       ↓
Add & LayerNorm
```

## Cross Attention

作用：生成内容时参考输入内容。

- **Q 来自 Decoder**（"我要生成什么"）
- **K、V 来自 Encoder**（"输入内容是什么"）

例如翻译 "I love AI" → "我爱人工智能" 时，Decoder 生成每个中文词都会参考 Encoder 的英文语义。

---

# 十、Mask 机制

## 为什么需要 Mask？

生成 "我 爱 人 工 智 能" 时，预测 "工" 不能提前看到 "智""能"，否则就是"作弊"。

## Causal Mask（因果掩码）

用**上三角矩阵**实现：将未来位置的注意力分数设为 $-\infty$，softmax 后变为 0。

```text
位置    我  爱  人  工  智  能
我      ✓   ✗   ✗   ✗   ✗   ✗
爱      ✓   ✓   ✗   ✗   ✗   ✗
人      ✓   ✓   ✓   ✗   ✗   ✗
工      ✓   ✓   ✓   ✓   ✗   ✗
```

这也是 GPT 能够**自回归生成文本**的关键。

---

# 十一、输出层

Decoder 最终输出经过：

```text
Linear（映射到词表大小）→ Softmax（概率分布）→ 预测下一个 token
```

例如：

```text
P("world") = 0.8, P("the") = 0.1, P("a") = 0.05, ...
```

---

# 十二、原论文关键超参数

| 超参数 | 值 | 说明 |
|---|---|---|
| d_model | 512 | 模型维度 |
| h（注意力头数） | 8 | Multi-Head 的头数 |
| d_k = d_v | 64 | 每个头的维度（512 / 8） |
| d_ff | 2048 | FFN 中间层维度（d_model × 4） |
| Encoder 层数 N | 6 | |
| Decoder 层数 N | 6 | |
| Dropout | 0.1 | |
| 训练数据 | WMT 2014（英德/英法） | 约 450 万句对 |
| 优化器 | Adam | β₁=0.9, β₂=0.98, ε=10⁻⁹ |

---

# 十三、训练技巧

## 1. Label Smoothing（标签平滑）

原论文使用 ε = 0.1，不把全部概率给正确答案，而是分配一小部分给其他词。

作用：防止模型过度自信，提升泛化能力。

## 2. Warmup + Inverse Square Root 学习率

```text
lr = d_model^(-0.5) × min(step^(-0.5), step × warmup_steps^(-1.5))
```

前 `warmup_steps` 步学习率**线性增长**，之后按**平方根衰减**。

> [!tip] 面试要点
> 训练初期参数随机、梯度不稳定，大学习率容易发散。先用小学习率"热身"，稳定后再逐步增大。

## 3. Teacher Forcing

Decoder 训练时，输入使用**真实的上一个 token**（而非模型自己预测的）。

```text
训练时：输入"我 爱" → 预测"人"（正确答案"人"作为下一步输入）
推理时：输入"我 爱" → 预测"人" → 输入"人" → 预测"工"
```

> [!warning] 训练和推理的不一致
> Teacher Forcing 导致训练时看到"正确前文"，推理时是"自己的预测"，这种 **exposure bias** 是已知问题。

---

# 十四、Pre-Norm vs Post-Norm

| 方式 | 结构 | 特点 |
|---|---|---|
| Post-Norm（原论文） | $x + F(x) \rightarrow LayerNorm$ | 表达能力强，但深层训练不稳定，需要 warmup |
| Pre-Norm（现代主流） | $x + F(LayerNorm(x))$ | 训练更稳定，GPT-2/Llama 等采用 |

> [!note] 面试延伸
> 现代大模型基本都用 Pre-Norm（或其变体如 RMSNorm），因为深层网络（几十到上百层）用 Post-Norm 很难训练。

---

# 十五、Transformer 的优势

| 对比 | RNN | Transformer |
|---|---|---|
| 计算方式 | 串行（必须按顺序） | 并行（整个序列同时计算） |
| 长距离依赖 | 差（梯度消失、记忆衰减） | 强（任意 token 直接建立关系） |
| GPU 利用率 | 低 | 高 |
| 训练速度 | 慢 | 快 |
| 扩展能力 | 差 | 强（适合大规模参数训练） |

---

# 十六、Transformer 的缺点

Self-Attention 的时间复杂度：**O(n²)**（每个 token 都要和其它 token 计算关系），长文本显存消耗大。

优化方案：

| 方案 | 思路 |
|---|---|
| FlashAttention | IO 感知的精确注意力，不改变结果，2~4x 加速 |
| Sparse Attention | 只计算部分 token 对的关系 |
| Linear Attention | 用核函数近似 softmax，复杂度降到 O(n) |
| Sliding Window Attention | Mistral 等采用，只关注局部窗口 |

---

# 十七、BERT、GPT、T5 区别

| 模型 | 结构 | 训练目标 | 核心能力 | 擅长任务 |
|---|---|---|---|---|
| BERT | Encoder Only | MLM + NSP（双向） | 理解 | 文本分类、NER、语义匹配 |
| GPT | Decoder Only | 下一个词预测（单向） | 生成 | 对话、写作、代码生成 |
| T5 | Encoder + Decoder | Span Corruption | 理解+生成 | 翻译、摘要、问答 |

> [!tip] 面试类比
> - GPT：写作文的人（续写大师）
> - BERT：做阅读理解的人（理解高手）
> - T5：全能语言老师（Text-to-Text 统一范式）

---

# 十八、Transformer 与现代大模型

现代大模型几乎全部建立在 Transformer 架构之上：

| 模型 | 基于 | 关键特点 |
|---|---|---|
| GPT 系列 | Decoder-only | 自回归生成，In-context Learning |
| BERT | Encoder-only | 双向理解，适合分类/检索 |
| Llama | Decoder-only | RoPE + RMSNorm + SwiGLU |
| Qwen | Decoder-only | 中文优化，GQA |
| Gemini | Decoder-only | 原生多模态 |

本质都是 Transformer 的变种。

---

# 十九、面试高频问题

## Q1：Transformer 为什么比 RNN 强？

> Transformer 利用 Attention 直接建立任意位置之间的联系，不再依赖逐步传递信息，因此具有更强的长距离建模能力和并行计算能力。

## Q2：为什么要除以 $\sqrt{d_k}$？

> 防止 $QK^T$ 数值过大，导致 softmax 梯度消失。除以 $\sqrt{d_k}$ 使方差稳定在 1 附近。

## Q3：为什么 GPT 只保留 Decoder？

> 生成任务只需要"预测下一个 Token"，Decoder 的因果掩码天然适配自回归生成。

## Q4：为什么 BERT 只保留 Encoder？

> 理解任务需要双向上下文，Encoder 的双向注意力可以同时看到前文和后文。

## Q5：Multi-Head Attention 的作用？

> 从多个子空间学习不同语义关系（语法、指代、语义等），增强模型表达能力。增加头数不增加总计算量。

## Q6：Attention 和 FFN 分别负责什么？

> Attention 负责 Token **之间**的信息交换，FFN 负责每个 Token **内部**的非线性变换。

## Q7：Pre-Norm 和 Post-Norm 的区别？

> Post-Norm 先 Add 再 Norm（原论文），表达能力强但深层训练不稳定。Pre-Norm 先 Norm 再 Add（现代主流），训练更稳定。

---

# 二十、最终总结（背诵版）

Transformer 是 Google 在 2017 年提出的序列建模架构，其核心思想是利用 Self-Attention 替代 RNN，实现全局信息交互。整体由 Encoder 和 Decoder 组成，核心模块包括 Multi-Head Attention、Position Encoding、Feed Forward、Residual 和 LayerNorm。相比 RNN，Transformer 具有更强的长距离依赖建模能力和并行计算能力。目前主流大模型如 GPT、BERT、Llama、Qwen 等都建立在 Transformer 架构或其变体之上，因此 Transformer 被认为是现代大模型的基础架构。
