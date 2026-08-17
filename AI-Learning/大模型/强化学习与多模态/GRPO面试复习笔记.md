# GRPO在大语言模型强化学习微调中的面试复习笔记

---

# 一、整体结构

## 1.1 大模型训练流程全景

```
Pretrain（预训练）
    ↓  学习语言知识，但输出风格不可控
SFT（监督微调）
    ↓  学会遵循指令，但质量未经优化
RLHF（人类反馈强化学习）
    ↓  用人类偏好对齐模型行为
PPO（近端策略优化）
    ↓  经典RLHF算法，但训练开销巨大
GRPO（组相对策略优化）
    ↓  DeepSeek提出，去掉Critic网络，降低训练成本
```

## 1.2 每个阶段解决什么问题

| 阶段 | 解决的问题 | 局限性 |
|------|-----------|--------|
| Pretrain | 学习语言知识、世界知识 | 无法遵循指令，输出不可控 |
| SFT | 学会按格式回答，遵循指令 | 容易过拟合标注数据，无法自我改进 |
| RLHF | 用人类偏好信号训练模型 | 需要大量人工标注，成本高 |
| PPO | 用RL优化模型输出质量 | 需要Critic网络，显存/算力翻倍 |
| **GRPO** | **去掉Critic，降低RL训练成本** | **advantage估计方差较大** |

## 1.3 为什么DeepSeek提出GRPO

DeepSeek在训练DeepSeekMath模型时面临的核心问题：

1. **数学推理需要强化学习**：SFT后的模型能解简单题，但复杂推理链需要RL优化
2. **PPO太贵**：3B模型的PPO训练需要同时加载Policy + Value两个模型，显存需求翻倍
3. **数学任务有天然的规则奖励**：不需要训练Reward Model，可以直接用程序验证答案
4. **Group Sampling天然适配**：对同一个数学题生成多个解法，组内比较即可估计优势

**一句话总结**：GRPO = PPO - Critic + Group Relative Advantage

---

# 二、GRPO整体流程

## 2.1 完整流程图（对应本项目代码）

```
Step 1: 从数据集采样Prompt
        例如：「使用数字 [3, 7, 12, 5]，创建一个等于 24 的等式」
        文件：train.py → CountdownTasksDataset
        ↓
Step 2: Policy Model生成多个Response（Group Sampling）
        对每个prompt生成G=8条response
        文件：grpo.py → rollout()
        ↓
Step 3: Reward Function评分
        对每条response计算reward（格式+答案正确性）
        文件：countdown_task.py → reward_function()
        ↓
Step 4: Group Relative Advantage计算
        组内reward标准化：A_i = (r_i - mean(r)) / std(r)
        文件：grpo.py → normalize_rewards_per_group()
        ↓
Step 5: GRPO Loss计算
        loss = -log_probs * advantages
        文件：grpo.py → update_policy()
        ↓
Step 6: Backward + 参数更新
        梯度累积 → 梯度裁剪 → AdamW更新
        文件：grpo.py → update_policy()
```

## 2.2 各文件职责对应

| 文件 | 对应GRPO步骤 | 核心函数 |
|------|-------------|---------|
| `train.py` | 训练主循环，串联所有步骤 | `main()` |
| `grpo.py` | Step 2+4+5+6：rollout、归一化、loss、更新 | `rollout()`, `normalize_rewards_per_group()`, `update_policy()` |
| `countdown_task.py` | Step 1+3：数据加载、reward计算 | `CountdownTasksDataset`, `reward_function()` |
| `data_types.py` | 数据结构定义 | `Episode`, `MiniBatch` |
| `qwen2_model.py` | 模型架构（Qwen2.5） | `Transformer` |
| `tokenizer.py` | 分词器 | `Tokenizer` |

## 2.3 一次GRPO Iteration的数据流

```
batch = {prefix: ["问题1", "问题2", ..., "问题32"]}  # 32个prompt
    ↓
rollout(): 32 × 8 = 256条轨迹
    episodes = [
        Episode(prefix="问题1", generated_token_ids=[...], reward=0.8),
        Episode(prefix="问题1", generated_token_ids=[...], reward=0.1),
        ...  # 共256个Episode
    ]
    ↓
normalize_rewards_per_group(): 按prefix分组，组内标准化
    episodes[0].reward = (0.8 - 0.5) / 0.3 = 1.0   # advantage
    episodes[1].reward = (0.1 - 0.5) / 0.3 = -1.33  # advantage
    ↓
update_policy(): 微批次梯度累积 → loss.backward()
    ↓
optimizer.step(): 参数更新
```

---

# 三、GRPO核心思想

## 3.1 为什么需要Group Sampling

### 核心问题：没有Critic，如何估计Advantage？

在PPO中，Advantage = Q(s,a) - V(s)，需要一个Critic网络V(s)来估计"期望回报"。

GRPO的解决方案：**用同一prompt下的多个response进行相对比较**。

### Group Sampling的工作原理

对同一个prompt生成G条response：

```
Prompt: 「使用数字 [3, 7, 12, 5] 创建等于24的等式」

Response 1: "让我一步步来思考... (3+7)×(12-5)..." → reward=1.0 ✓
Response 2: "让我一步步来思考... 3×7+12-5..."     → reward=0.1 ✗
Response 3: "让我一步步来思考... (12-5)×(3+7)..." → reward=1.0 ✓
Response 4: "让我一步步来思考... 3×12-7-5..."     → reward=0.0 ✗
Response 5: "让我一步步来思考... (12+7)×3-5..."   → reward=0.0 ✗
Response 6: "让我一步步来思考... (12-5)×3+7..."   → reward=1.0 ✓
Response 7: "让我一步步来思考... 7×3+12-5..."     → reward=0.1 ✗
Response 8: "让我一步步来思考... 12×(7-5)+3..."   → reward=1.0 ✓
```

组内reward = [1.0, 0.1, 1.0, 0.0, 0.0, 1.0, 0.1, 1.0]
mean = 0.525, std = 0.47

```
Response 1 的advantage = (1.0 - 0.525) / 0.47 = +1.01  → 增大概率
Response 2 的advantage = (0.1 - 0.525) / 0.47 = -0.90  → 减小概率
Response 4 的advantage = (0.0 - 0.525) / 0.47 = -1.12  → 减小概率
```

### 为什么这样做有效？

1. **同一prompt的response共享上下文**：差异仅来自生成内容，可以公平比较
2. **组内标准化消除尺度差异**：不同prompt的reward尺度可能不同（有的题简单，整体reward高）
3. **正负advantage都有用**：正的增大概率，负的减小概率，都在优化策略

### Group Size G的影响

| G值 | 优势 | 劣势 | 适用场景 |
|-----|------|------|---------|
| G=4 | 计算快 | advantage噪声大 | 快速实验 |
| **G=8** | **平衡点** | - | **默认选择** |
| G=16 | advantage更准确 | 计算量×2 | 追求精度 |
| G=32 | advantage非常稳定 | 计算量×4 | 大规模训练 |

论文实验表明 **G=8** 是较好的平衡点。

### 面试回答模板

> **Q: GRPO的Group Sampling有什么作用？**
>
> A: GRPO的核心创新是用组内相对比较替代Critic网络。对同一个prompt生成G条response，它们共享相同的上下文（prompt），只有生成内容不同。因此可以用组内reward的相对高低来判断哪条response更好，而不需要一个额外的Value网络来估计"期望回报"。Group Size G越大，advantage估计越准确，但计算开销也越大。实验表明G=8是较好的平衡点。

---

# 四、GRPO Advantage详解

## 4.1 PPO的Advantage计算

```
A_PPO(s, a) = Q(s, a) - V(s)
```

- Q(s, a)：在状态s执行动作a后的期望回报（通过Monte Carlo采样估计）
- V(s)：状态s的价值（由Critic网络输出）
- 需要**额外训练一个与Policy同等规模的Critic网络**

### 为什么PPO需要Critic？

在标准RL中，reward是稀疏的（只有游戏结束才知道好不好）。
Critic网络学习估计"从当前状态开始，未来能获得多少总reward"，
从而为每个动作提供即时反馈。

## 4.2 GRPO的Advantage计算

```
A_GRPO(i) = (r_i - mean(r_group)) / (std(r_group) + ε)
```

- r_i：第i条response的reward
- mean(r_group)：组内所有response的reward均值
- std(r_group)：组内所有response的reward标准差
- ε = 1e-4：防止除零

### 每个符号的含义

| 符号 | 含义 | 在代码中的位置 |
|------|------|--------------|
| r_i | 单条response的reward | `episode.reward`（原始值） |
| mean(r) | 组内reward均值 | `np.mean(group_rewards)` |
| std(r) | 组内reward标准差 | `np.std(group_rewards)` |
| A_i | 标准化后的advantage | `normalized_reward`（替换后的值） |

### 为什么Reward要标准化？

1. **消除prompt难度差异**：有的题整体简单（reward都高），有的题整体难（reward都低）
2. **统一尺度**：标准化后advantage的均值为0、方差为1
3. **梯度稳定**：避免不同prompt的reward尺度差异导致梯度不稳定

### 为什么可以替代Critic？

关键洞察：**Critic的本质是估计"这个action有多好"**。

GRPO的替代方案：**在同一个prompt下，"这个response比其他response好多少"**。

两种方式得到的信息类似：
- Critic提供的是"绝对价值"（这个action值多少分）
- GRPO提供的是"相对价值"（这个response比组内平均好多少）

对于优化策略来说，**相对价值已经足够**——我们只需要知道"应该增大还是减小这个response的概率"。

### 优点与缺点

| 维度 | 优点 | 缺点 |
|------|------|------|
| 资源 | 不需要Critic网络，显存/计算量减半 | - |
| 实现 | 实现简单，不需要维护Critic | - |
| 稳定性 | - | advantage估计方差较大 |
| 精度 | - | 依赖Group Size |
| 适用性 | - | 需要有明确的reward信号 |

### 面试回答模板

> **Q: 为什么GRPO不需要Critic网络？**
>
> A: GRPO用"组内相对比较"替代了Critic的价值估计。对同一个prompt生成G条response，通过组内reward的标准化得到advantage。Critic的本质是估计"这个action有多好"，GRPO的替代方案是"这个response比组内其他response好多少"。两种方式得到的信息类似，但GRPO省去了一个同等规模的Critic网络，训练资源需求减半。代价是advantage估计方差较大，但通过增大Group Size可以缓解。

---

# 五、GRPO数学原理

## 5.1 Policy Gradient基础

### 标准策略梯度公式

```
∇J(θ) = E_{τ~πθ} [Σ_t ∇log πθ(a_t|s_t) * G_t]
```

| 符号 | 含义 | LLM对应 |
|------|------|---------|
| πθ | 策略函数（Actor） | 语言模型本身 |
| a_t | 动作 | 生成的下一个token |
| s_t | 状态 | 已生成的token序列（prompt + 之前的token） |
| G_t | 从t开始的累积回报 | 该token对最终reward的贡献 |
| τ | 一条完整轨迹 | prompt + 完整response |

### LLM中的对应关系

- **Token就是Action**：模型在每个位置选择一个token
- **Sequence就是Trajectory**：从prompt开始到EOS结束的完整序列
- **Reward是稀疏的**：只有整个response生成完毕才能打分

## 5.2 PPO目标函数

```
L_PPO = E[min(ratio * A, clip(ratio, 1-ε, 1+ε) * A)]
```

### Policy Ratio

```
ratio = πθ_new(a|s) / πθ_old(a|s)
```

- πθ_new：当前策略（正在被优化的模型）
- πθ_old：采样时的策略（生成response时的模型）
- ratio > 1 → 当前策略比旧策略更倾向于选择这个action
- ratio < 1 → 当前策略比旧策略更不倾向于选择这个action

### Clip机制

```
clip(ratio, 1-ε, 1+ε)
```

- ε通常取0.1-0.2
- 当ratio > 1+ε时，强制裁剪到1+ε（防止策略更新过大）
- 当ratio < 1-ε时，强制裁剪到1-ε（防止策略更新过小）

**为什么要clip？**

没有clip的话，如果某个action的advantage很大（A>>0），ratio会被推得很大，
导致策略更新幅度过大，训练不稳定。clip机制提供了"信任域约束"。

### GRPO为什么继承PPO思想

GRPO继承了PPO的ratio和clip机制，但**去掉了Critic网络**：
- PPO: A = Q(s,a) - V(s)  → 需要Critic
- GRPO: A = (r - mean(r)) / std(r)  → 不需要Critic

## 5.3 GRPO完整Loss（论文版本）

```
L_GRPO = E_q [ E_{o~πθ(q)} [ min(ρ·Â, clip(ρ, 1-ε, 1+ε)·Â) ] - β·KL(πθ || πref) ]
```

| 组件 | 公式 | 作用 |
|------|------|------|
| ρ (ratio) | πθ(o\|q) / πold(o\|q) | 策略比率，衡量新旧策略的差异 |
| Â (advantage) | (r - mean(r)) / std(r) | 组内标准化优势 |
| clip(ρ) | clip(ρ, 1-ε, 1+ε) | 信任域约束 |
| KL(πθ\|\|πref) | Σ πθ log(πθ/πref) | KL散度惩罚，防止偏离参考模型 |
| β | KL惩罚系数 | 控制KL约束的强度 |

### 本代码的loss实现

```python
# 代码实际实现
obj = log_probs * batch_advantages[:, None]
loss = -obj.mean()
```

```math
L = -\frac{1}{N} \sum_t \log \pi_\theta(o_t | o_{<t}, q) \cdot A_i
```

### 为什么代码形式与论文不同？——Single-pass推导

本实现对每条轨迹只更新一次策略（single-pass），在 θ=θ_old 处求梯度：

```
ρ = πθ / πθ_old = 1       （θ=θ_old时，新旧策略相同）
clip(1, 1-ε, 1+ε) = 1     （ratio=1时，clip不生效）
min(ρ·A, clip(ρ)·A) = A   （两项相等，min无影响）
```

梯度推导：

```
∇θ min(ρ·A, clip(ρ)·A)|θ=θold
  = ∇θ (ρ·A)|θ=θold                    （clip在ρ=1处不生效）
  = A · ∇θ (πθ/πθ_old)|θ=θold
  = A · ∇θ πθ|θ=θold / πθ_old
  = A · ∇θ log πθ|θ=θold               （log梯度技巧）
```

因此 **loss = -log_probs × advantage** 在 single-pass 条件下与论文公式**数学等价**。

**Ratio和Clip不是"被省略"，而是在 θ=θ_old 处退化为无影响。**

### 真正缺失的组件

| 组件 | 状态 | 说明 |
|------|------|------|
| ratio | ✅ 数学等价 | θ=θ_old时ρ=1，乘不乘都一样 |
| clip | ✅ 数学等价 | ρ=1时clip不生效 |
| KL惩罚 | ⚠️ 真正缺失 | 本实现未添加KL(πθ\|\|πref) |

**KL惩罚缺失的影响**：策略可能偏离参考模型太远，导致生成质量下降。
但对于规则奖励明确的数学任务，不加KL也能稳定训练。

### 面试回答模板

> **Q: 你的GRPO代码和论文有什么区别？**
>
> A: 代码形式上看起来省略了ratio和clip，但实际上是因为本实现对每条轨迹只更新一次（single-pass），在θ=θ_old处求梯度。此时ratio=1，clip不生效，所以loss = -log_probs × advantage 与论文公式数学等价。真正缺失的是KL惩罚项（β·KL(πθ||πref)），这是一个设计选择——对于规则奖励明确的数学任务，不加KL也能稳定训练。

---

# 六、Reward设计

## 6.1 本项目的Reward设计

```python
# countdown_task.py → reward_function()
reward = 0.1 * format_reward + answer_reward
```

### Format Reward（格式奖励）

```
预期格式：
<think>...推理过程...
</think>
<answer>数学表达式</answer>
```

| 匹配情况 | 奖励 |
|---------|------|
| 完全匹配（think在前，answer在后） | 1.0 |
| 有<think>标签 | +0.1 |
| 有<answer>标签 | +0.5 |
| 都没有 | 0.0 |

### Answer Reward（答案奖励）

| 条件 | 奖励 |
|------|------|
| 答案正确（使用所有数字，每个一次，结果等于目标） | 1.0 |
| 任一条件不满足 | 0.0 |

### 奖励尺度分析

| 场景 | format_reward | answer_reward | 总reward |
|------|--------------|---------------|---------|
| 格式对 + 答案对 | 1.0 | 1.0 | **1.1** |
| 格式对 + 答案错 | 1.0 | 0.0 | **0.1** |
| 格式错 + 答案对 | 0.0 | 1.0 | **1.0** |
| 格式错 + 答案错 | 0.0 | 0.0 | **0.0** |

**设计考量**：format权重只有0.1，确保答案正确性是主要优化目标。

## 6.2 规则Reward vs Reward Model

| 维度 | 规则Reward | Reward Model |
|------|-----------|-------------|
| 准确性 | 精确（程序验证） | 有噪声（模型预测） |
| 成本 | 无需训练 | 需要额外训练 |
| 适用场景 | 有明确验证规则的任务（数学、代码） | 开放式任务（对话、写作） |
| 灵活性 | 低（规则固定） | 高（可学习复杂偏好） |
| 本项目 | ✅ 使用规则Reward | - |

### 什么时候用什么？

- **数学推理**：规则Reward（答案对错可以精确验证）
- **代码生成**：规则Reward（测试用例通过率）
- **对话质量**：Reward Model（需要学习人类偏好）
- **创意写作**：Reward Model（没有客观标准）

### 面试回答模板

> **Q: 你的Reward函数是怎么设计的？**
>
> A: 本项目使用规则奖励，包含两部分：format_reward（0.1权重）检查输出是否遵循think/answer格式，answer_reward（1.0权重）验证答案正确性。总reward = 0.1 * format + accuracy。格式奖励权重低，确保答案正确性是主要优化目标。对于数学推理任务，规则奖励比训练的Reward Model更准确（程序验证无噪声），且不需要额外训练。

---

# 七、GRPO代码实现解析

## 7.1 数据结构

### Episode（一条轨迹）

```python
@dataclass
class Episode:
    prefix: str              # prompt文本
    text: str                # prompt + response完整文本
    prefix_token_ids: List[int]  # prompt的token ids
    prefix_tokens: List[str]     # prompt的token文本列表
    generated_token_ids: List[int]  # response的token ids
    is_finished: bool        # 是否正常结束（遇到EOS）
    reward: float            # reward/advantage（rollout时是reward，之后被替换为advantage）
    reward_info: Dict[str, float]  # 详细奖励信息
```

**关键点**：`reward`字段在rollout阶段存储原始reward，经过`normalize_rewards_per_group()`后被替换为advantage。

### MiniBatch（一批prompt）

```python
@dataclass
class MiniBatch:
    prefix: List[str]           # 一批prompt文本
    prefix_tokens: List[List[str]]
    prefix_token_ids: List[List[int]]
    numbers: List[List[int]]    # 数字列表（用于reward验证）
    target: List[int]           # 目标答案（用于reward验证）
```

## 7.2 Rollout阶段

### 核心流程

```python
# grpo.py → rollout()
for cur_pos in range(min_prompt_len, total_len):
    # 1. 模型前向推理
    logits = model.inference(tokens[:, prev_pos:cur_pos], prev_pos)
    
    # 2. logits → 概率分布
    probs = torch.softmax(logits[:, -1], dim=-1)
    
    # 3. 多项式采样（不是argmax）
    next_token = torch.multinomial(probs, num_samples=1)
    
    # 4. 处理已结束的序列
    next_token = torch.where(is_finished, pad_token_id, next_token)
```

### 采样策略

- **多项式采样**（`torch.multinomial`）：从概率分布中随机采样
- **不是argmax**：argmax会退化为贪心解码，response多样性丧失
- **多样性对GRPO很重要**：如果所有response都一样，组内比较就没有意义

### 并行推理的工程挑战

1. **不同长度的prompt**：padding到相同长度
2. **不同结束时间的response**：用mask标记已结束的序列
3. **KV Cache**：避免重复计算，加速自回归生成

## 7.3 Advantage计算

```python
# grpo.py → normalize_rewards_per_group()
for group in groups.values():
    group_rewards = [item.reward for item in group]
    mean_reward = np.mean(group_rewards)
    std_reward = np.std(group_rewards)
    
    for episode in group:
        normalized_reward = (episode.reward - mean_reward) / (std_reward + 1e-4)
        episode = dataclasses.replace(episode, reward=normalized_reward)
```

**关键细节**：
- `dataclasses.replace()`创建新Episode，而非原地修改
- `1e-4`是防止除零的微小常数

## 7.4 Loss计算

```python
# grpo.py → update_policy()

# 1. 构建input/target（右移一位）
input_token_ids = batch_token_ids[:, :-1]   # [prompt + response] 去掉最后一个
target_token_ids = batch_token_ids[:, 1:]    # [prompt + response] 去掉第一个

# 2. 前向传播
logits = model.forward(input_token_ids).float()

# 3. 计算log πθ(o|q)
log_probs = -torch.nn.functional.cross_entropy(
    logits.reshape(-1, logits.size(-1)),
    target_token_ids.reshape(-1),
    ignore_index=pad_token_id,
    reduction="none",
).reshape(input_token_ids.shape[0], -1)

# 4. 计算loss = -log_probs * advantages
obj = log_probs * batch_advantages[:, None]
obj = (obj * target_masks).sum() / num_target_tokens
loss = -obj

# 5. 反向传播（梯度累积）
loss.backward()
```

### 为什么用交叉熵计算log_probs？

```python
# 数学等价关系
cross_entropy(logits, target) = -log(p(target))
所以 -cross_entropy = log(p(target)) = log πθ(o_t|o_{<t}, q)
```

### Mask的设计

```python
batch_masks = [
    [0] * len(prefix_token_ids)      # prompt部分 → 不参与loss
    + [1] * len(generated_token_ids)  # response部分 → 参与loss
    + [0] * padding                   # padding → 不参与loss
]
```

**为什么只对response部分计算loss？** prompt是条件输入，不需要优化；只优化response的生成概率。

---

# 八、Qwen2模型结构补充

本项目使用Qwen2.5-3B作为基座模型，从零实现。

## 8.1 RMSNorm

```
y = x / √(mean(x²) + ε) * γ
```

- 替代LayerNorm，省略了均值中心化和偏置项β
- 计算更简单，效果相当
- 现代LLM的标准选择

## 8.2 RoPE（旋转位置编码）

```
q_m = R(θ, m) * q
k_n = R(θ, n) * k
q_m · k_n = q · R(θ, m-n) * k  # 只依赖相对位置
```

- 将位置信息编码为旋转矩阵
- 支持相对位置注意力
- 外推性好（可处理更长序列）

## 8.3 GQA（分组查询注意力）

```
H = 16（Q头数）
H_kv = 2（KV头数）
G = H / H_kv = 8（每8个Q头共享一组KV）
```

- 大幅减少KV Cache显存占用
- 对GRPO的Rollout阶段至关重要（256条序列并行生成）

## 8.4 SwiGLU激活函数

```
FFN(x) = W2 * (SiLU(W_gate * x) ⊙ W_up * x)
```

- 门控机制：gate控制信息流通
- SiLU平滑：比ReLU更适合梯度传播
- 实验表明效果优于标准ReLU FFN

---

# 九、PPO vs GRPO重点对比

| 维度 | PPO | GRPO |
|------|-----|------|
| **Critic网络** | 需要（与Policy同等规模） | **不需要** |
| **Advantage计算** | A = Q(s,a) - V(s) | **A = (r - mean(r)) / std(r)** |
| **训练资源** | 2x（Policy + Critic） | **1x（仅Policy）** |
| **显存占用** | 高（加载两个模型） | **低（只加载一个模型）** |
| **实现复杂度** | 较高（需维护Critic） | **较低** |
| **训练稳定性** | 较好（Critic提供稳定估计） | 依赖Group Size |
| **Advantage方差** | 较小（Critic学习平滑） | **较大（依赖采样）** |
| **适用场景** | 通用RL任务 | **数学推理等有明确reward的任务** |
| **工程复杂度** | 高（Critic训练、更新同步） | **低** |
| **DeepSeek选择** | - | **✅ 选择GRPO** |

### DeepSeek为什么选择GRPO？

1. **显存瓶颈**：训练3B模型时，PPO需要同时加载Policy + Value（6B参数），GRPO只需要3B
2. **数学任务天然适配**：有明确的规则奖励，不需要RM
3. **Group Sampling效率高**：对同一个prompt生成8条response，可以并行处理
4. **实验效果好**：DeepSeekMath在GSM8K上达到了SOTA

---

# 十、面试高频问题

## 基础概念

### Q1: GRPO是什么？

**A**: GRPO（Group Relative Policy Optimization）是DeepSeek提出的强化学习算法，核心创新是用"组内相对比较"替代PPO中的Critic网络。对同一个prompt生成G条response，通过组内reward的标准化得到advantage，从而省去了Value网络的训练。

**追问**: GRPO全称是什么？
**A**: Group Relative Policy Optimization。

### Q2: 为什么提出GRPO？

**A**: PPO训练需要同时加载Policy和Critic两个模型，显存/计算量翻倍。对于大模型（3B+），这成为严重的资源瓶颈。GRPO去掉了Critic，只用Policy模型，训练资源需求减半。

**追问**: 主要解决了什么痛点？
**A**: (1) 显存占用：PPO需要Policy+Critic两个模型；(2) 计算成本：Critic的训练和推理开销；(3) 实现复杂度：需要维护Critic的更新同步。

### Q3: GRPO的核心思想是什么？

**A**: 对同一个prompt生成一组（Group）response，通过组内reward的相对排名来估计优势（Advantage），从而完全移除Critic网络。

### Q4: GRPO和PPO有什么区别？

**A**: 最核心的区别是GRPO去掉了Critic网络。PPO用Critic估计V(s)来计算Advantage = Q(s,a) - V(s)，GRPO用组内reward标准化 A = (r - mean(r)) / std(r)。GRPO的训练资源需求减半，但advantage估计方差较大。

## 数学原理

### Q5: GRPO的Advantage怎么计算？

**A**: 对同一个prompt的G条reward做标准化：A_i = (r_i - mean(r)) / (std(r) + ε)。正的advantage表示该response比组内平均好，应增大概率；负的表示比平均差，应减小概率。

**追问**: 为什么要标准化？
**A**: (1) 消除prompt难度差异（有的题简单，整体reward高）；(2) 统一尺度，使不同prompt的advantage可比；(3) 梯度稳定，避免reward尺度差异导致训练不稳定。

### Q6: Policy Ratio是什么？

**A**: ratio = πθ_new(o|q) / πθ_old(o|q)，表示新策略与旧策略在同一条response上的概率比。ratio > 1表示新策略更倾向于选择这个response，ratio < 1表示更不倾向于。

### Q7: Clip机制的作用？

**A**: clip(ratio, 1-ε, 1+ε)限制了策略更新的幅度。当ratio偏离1太远时（即新旧策略差异过大），强制裁剪到[1-ε, 1+ε]范围内。这防止了策略更新过大导致的训练不稳定。

### Q8: KL散度惩罚的作用？

**A**: KL(πθ||πref)惩罚策略偏离参考模型（通常是SFT后的模型）太远。防止策略为了追求高reward而"走捷径"（reward hacking），保持生成质量。

### Q9: 为什么GRPO可以不用Critic？

**A**: Critic的本质是估计"这个action有多好"。GRPO用"这个response比组内其他response好多少"来替代。两种方式得到的信息类似，但GRPO更简单——不需要额外的网络，只需要组内比较。

**追问**: 这样做的trade-off是什么？
**A**: advantage估计方差较大（依赖Group Size），但通过增大G可以缓解。实验表明G=8已足够稳定。

### Q10: GRPO的loss公式是什么？

**A**: 论文版本：L = -min(ratio*A, clip(ratio)*A) + β*KL。本代码：L = -log_probs * A。在single-pass（每条轨迹只更新一次）条件下，θ=θ_old时ratio=1、clip不生效，两者数学等价。真正缺失的是KL惩罚项。

### Q10b: 为什么代码中没有ratio和clip，却说与论文等价？

**A**: 本实现对每条轨迹只更新一次策略（single-pass），在θ=θ_old处求梯度。此时：

```
ρ = πθ/πθ_old = 1（新旧策略相同）
clip(1, 1-ε, 1+ε) = 1（ratio=1时不裁剪）
min(ρ·A, clip(ρ)·A) = A（两项相等）
```

用log梯度技巧推导：∇θ(ρ·A)|θ=θold = A·∇θ log πθ|θ=θold

所以 loss = -log_probs × advantage 与论文公式数学等价。ratio和clip不是"被省略"，而是在θ=θ_old处退化为无影响。只有multi-step更新（多次前向+反向后再同步旧策略）时，θ偏离θ_old，clip才真正生效。

## 工程实现

### Q11: Group Size如何选择？

**A**: Group Size G是GRPO的核心超参数。G越大，advantage估计越准确，但计算开销越大。实验表明G=8是较好的平衡点。G=4适合快速实验，G=16适合追求精度。

### Q12: Reward不稳定怎么办？

**A**: (1) 增大Group Size，使advantage估计更稳定；(2) 使用reward normalization（本项目已实现）；(3) 梯度裁剪（max_grad_norm=1.0）；(4) 使用KL惩罚防止策略偏移。

### Q13: 什么是Micro-batch梯度累积？

**A**: 由于显存限制，不能一次性处理所有Episode。将Episode分成多个micro-batch，每个计算loss后backward累积梯度，最后统一更新参数。等价于大batch训练，但显存需求更小。

### Q14: 梯度裁剪为什么重要？

**A**: 大模型训练中，梯度可能突然爆炸（gradient explosion），导致参数更新过大。梯度裁剪（clip_grad_norm）将梯度L2范数限制在阈值内（本项目为1.0），防止训练崩溃。

### Q15: KV Cache在GRPO中的作用？

**A**: Rollout阶段需要对256条序列并行自回归生成。KV Cache避免了每生成一个token都重新计算所有位置的Key和Value，将推理复杂度从O(n²)降低到O(n)。GQA进一步减少了KV Cache的显存占用。

## 深入理解

### Q16: GRPO有什么缺点？

**A**: (1) advantage估计方差较大（依赖Group Size）；(2) 需要有明确的reward信号（不适合开放式任务）；(3) 本实现未添加KL惩罚，策略可能偏离参考模型。

### Q17: GRPO是否完全替代PPO？

**A**: 不是。GRPO适用于有明确reward的数学推理任务。对于开放式任务（对话、写作），仍需要PPO + Reward Model。GRPO是PPO的简化变体，不是替代品。

### Q18: 如果让你完善这个GRPO实现，你会加什么？

**A**: (1) 添加KL惩罚（需要Reference Model，这是当前真正缺失的组件）；(2) 支持multi-step更新（多次前向+反向后再同步旧策略，此时clip才真正生效）；(3) 添加学习率调度；(4) 添加early stopping。

### Q19: GRPO的entropy有什么用？

**A**: 熵衡量模型对自己预测的"确定程度"。熵大→模型在探索，熵小→模型收敛。训练中监控entropy可以发现训练不稳定（如entropy突然增大可能是reward hacking）。

### Q20: 为什么用规则Reward而不是Reward Model？

**A**: 数学任务可以用程序精确验证答案，规则奖励无噪声且不需要额外训练RM。对于开放式任务（对话、写作），没有客观标准，需要Reward Model学习人类偏好。

## 代码相关

### Q21: 你的GRPO代码流程是什么？

**A**: (1) 从数据集采样32个prompt；(2) 每个prompt生成8条response（rollout）；(3) 对每条response计算reward；(4) 组内reward标准化得到advantage；(5) 计算loss = -log_probs * advantage；(6) 梯度累积 + 梯度裁剪 + AdamW更新。

### Q22: Episode的reward字段什么时候被替换为advantage？

**A**: 在`normalize_rewards_per_group()`函数中。rollout阶段存储原始reward，该函数将其替换为标准化的advantage值。后续的`update_policy()`直接使用这个字段作为advantage。

### Q23: 为什么用交叉熵计算log_probs？

**A**: 数学等价：cross_entropy(logits, target) = -log(p(target))。所以 -cross_entropy = log(p(target)) = log πθ(o_t|o_{<t}, q)。这避免了额外实现log_softmax。

### Q24: Mask的设计有什么考量？

**A**: mask=1表示该位置是response token（参与loss计算），mask=0表示prompt或padding（不参与loss）。只优化response的生成概率，prompt作为条件输入不需要优化。

### Q25: 代码中为什么用dataclasses.replace？

**A**: `dataclasses.replace()`创建新Episode对象而非原地修改，避免副作用。在`normalize_rewards_per_group()`中，需要将reward替换为advantage但不修改原始Episode。

### Q26: 训练模式和推理模式的区别？

**A**: 训练模式（forward）：一次处理整个序列，使用causal mask，不需要KV Cache。推理模式（inference）：使用KV Cache，每次只处理新token，逐步自回归生成。

### Q27: tie_word_embeddings有什么用？

**A**: 共享embedding层和输出投影的权重，减少参数量。对于3B模型，这可以节省约30M参数。实验表明效果与不共享相当。

### Q28: checkpoint_sequential的作用？

**A**: 梯度检查点，用计算换显存。前向传播时不保存中间激活值，反向传播时重新计算。代价约30%额外计算，但显存占用大幅减少。

### Q29: 数据集怎么划分训练集和测试集？

**A**: parquet文件中，前N-test_size条为训练集，后test_size条为测试集。通过`iloc`索引切分，不使用随机划分（保证可复现）。

### Q30: DataLoader的collate_fn做了什么？

**A**: 将多个独立样本的字段分别收集到列表中，构建MiniBatch。例如32个prompt的prefix_token_ids收集为一个列表。

## 追问应对

### Q31: GRPO的训练稳定性如何保证？

**A**: (1) 梯度裁剪（max_grad_norm=1.0）；(2) Group Size足够大（G=8）；(3) reward normalization消除尺度差异；(4) 微批次梯度累积平滑梯度。完整实现还应添加clip和KL。

### Q32: 如果reward全是0或全是1怎么办？

**A**: 此时std(r)=0，advantage全为0或未定义。加了ε=1e-4可以避免除零，但advantage仍接近0，策略无法学习。解决方案：增大Group Size，或调整reward设计使其有区分度。

### Q33: GRPO适合什么类型的任务？

**A**: 有明确reward信号的任务：数学推理（答案对错可验证）、代码生成（测试用例通过率）、游戏（得分）。不适合开放式任务（对话、写作），因为难以设计客观reward。

### Q34: 为什么不用argmax采样？

**A**: argmax是贪心解码，所有response都一样，组内比较没有意义。多项式采样引入随机性，使response有差异，advantage才有意义。

### Q35: 训练时需要冻结哪些参数？

**A**: 本实现不冻结任何参数（全量微调）。实际中可能冻结embedding层（减少参数量），或冻结前几层（保留预训练知识）。

### Q36: 如何评估GRPO训练效果？

**A**: (1) 测试集上的answer_reward（正确率）；(2) format_reward（格式遵循度）；(3) entropy（训练稳定性）；(4) grad_norm（梯度健康度）。本项目每10步在测试集上评估一次。

### Q37: GRPO的收敛速度如何？

**A**: 比PPO快（因为每次更新更直接），但比SFT慢（需要生成大量response采样）。通常需要数百到数千个step。

### Q38: 如何处理长序列？

**A**: (1) 限制max_gen_len（本项目为1024）；(2) 使用GQA减少KV Cache显存；(3) 梯度检查点节省显存；(4) 微批次处理避免OOM。

### Q39: 本项目的超参数选择依据？

**A**: BATCH_SIZE=256、NUM_QUESTIONS=32、G=8：平衡了batch效率和显存。lr=1e-5：大模型微调的常用学习率。max_grad_norm=1.0：防止梯度爆炸的标准值。

### Q40: 如果让你从零实现GRPO，你会怎么设计？

**A**: (1) 先实现rollout（并行生成+KV Cache）；(2) 再实现reward函数（规则奖励）；(3) 然后实现advantage计算（组内标准化）；(4) 最后实现loss（policy gradient + clip + KL）。每步都用小规模实验验证正确性。

---

# 十一、项目面试介绍

> **请介绍一下你的GRPO项目**

我实现了一个基于DeepSeek GRPO算法的大模型强化学习微调项目，目标是让Qwen2.5-3B模型在数学推理任务上表现更好。

**项目背景**：GRPO是DeepSeek在DeepSeekMath论文中提出的强化学习算法，核心创新是用组内相对比较替代PPO中的Critic网络，训练资源需求减半。

**技术方案**：基座模型是Qwen2.5-3B，从零实现了完整的模型架构（RMSNorm、RoPE、GQA、SwiGLU）。训练任务是Countdown Task——给定一组数字和目标值，用四则运算构建等式。

**GRPO训练流程**：每步从数据集采样32个prompt，每个prompt生成8条response（Group Size=8），用规则奖励打分（format_reward 0.1权重 + answer_reward 1.0权重），然后在组内对reward标准化得到advantage，最后计算策略梯度loss并更新参数。

**Reward设计**：使用规则奖励而非Reward Model。format_reward检查输出是否遵循think/answer格式，answer_reward验证答案正确性。对于数学任务，规则奖励比RM更准确（程序验证无噪声）。

**与论文的差异**：本实现采用single-pass更新（每条轨迹只更新一次），此时ratio=1、clip不生效，loss = -log_probs × advantage 与论文公式数学等价。真正缺失的是KL惩罚项，但对于规则奖励明确的数学任务，不加KL也能稳定训练。

**自己的理解**：GRPO的核心洞察是"Critic的本质是估计action的价值，组内比较可以提供类似的信息"。这在数学推理任务中特别有效，因为同一prompt的多个解法可以直接比较优劣。

---

# 十二、学习总结

## 必须掌握（⭐⭐⭐⭐⭐）

| 知识点 | 重要性 | 掌握程度自测 |
|--------|--------|-------------|
| GRPO核心思想 | ⭐⭐⭐⭐⭐ | 能否用一句话解释GRPO？ |
| Advantage计算 | ⭐⭐⭐⭐⭐ | 能否写出公式并解释每个符号？ |
| Reward设计 | ⭐⭐⭐⭐⭐ | 能否解释format+accuracy的组合？ |
| PPO vs GRPO | ⭐⭐⭐⭐⭐ | 能否列出3个以上核心区别？ |

## 重要掌握（⭐⭐⭐⭐）

| 知识点 | 重要性 | 掌握程度自测 |
|--------|--------|-------------|
| Loss实现 | ⭐⭐⭐⭐ | 能否解释为什么用交叉熵？ |
| Sampling策略 | ⭐⭐⭐⭐ | 能否解释为什么不用argmax？ |
| 代码流程 | ⭐⭐⭐⭐ | 能否画出数据流？ |

## 了解即可（⭐⭐⭐）

| 知识点 | 重要性 | 掌握程度自测 |
|--------|--------|-------------|
| 模型结构 | ⭐⭐⭐ | 能否解释GQA的优势？ |
| KV Cache | ⭐⭐⭐ | 能否解释为什么能加速？ |
| 梯度检查点 | ⭐⭐⭐ | 能否解释trade-off？ |

## 面试前最后检查

- [ ] 能否在3分钟内介绍GRPO项目？
- [ ] 能否写出GRPO的advantage公式？
- [ ] 能否解释GRPO与PPO的3个核心区别？
- [ ] 能否说明本代码与论文的差异？
- [ ] 能否解释为什么GRPO不需要Critic？
- [ ] 能否解释Group Sampling的作用？
- [ ] 能否说明Rule Reward vs RM的适用场景？

---

*笔记生成时间：2026-07-24*
*基于项目：GRPO-Zero（Qwen2.5-3B + Countdown Task）*
