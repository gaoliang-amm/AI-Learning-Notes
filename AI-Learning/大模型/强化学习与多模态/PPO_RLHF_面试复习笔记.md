# PPO 在大语言模型 RLHF 中的面试复习笔记

> 基于 InstructGPT 项目代码整理，适用于大模型微调岗位面试准备

---

## 1. RLHF 整体流程

### 1.1 四阶段训练流水线

```
Pretrain (CLM)          在大规模语料上预训练，学习语言能力
     ↓
SFT (有监督微调)         在指令-回复对上微调，学习"按指令回复"
     ↓
Reward Model            训练奖励模型，学会评估回复质量
     ↓
PPO Optimization        用强化学习优化策略，生成高质量回复
```

**对应项目代码：**

| 阶段 | 代码文件 | 作用 |
|------|----------|------|
| 预训练 | `0-PRETRAIN.py` | 在中文祝福语料上从零训练 GPT-2 |
| SFT | `1-SFT.py` | 在电商评论数据上微调，教会模型"回复" |
| RM | `2-RM.py` | 用带标签评论训练奖励模型 |
| PPO | `3-PPO.py` | 用 RM 打分 + KL 惩罚 + PPO 优化策略 |

### 1.2 为什么预训练模型不能直接满足人类需求？

预训练模型的目标是 **next-token prediction**——学会的是"语言的统计规律"，而非"人类的偏好"。

具体问题：
- **不遵循指令**：给它一篇文章，它会续写，而不是回答问题
- **不安全**：可能生成有害、虚假、偏见内容
- **不 helpful**：回答可能冗长、离题、无意义
- **不 aligned**：模型优化的是 P(text)，而非 P(helpful|text)

### 1.3 为什么需要 SFT？

SFT 解决的是 **"格式"和"能力"** 问题：
- 教会模型"看到指令 → 生成回复"的模式
- 在任务数据上微调，激活预训练中学到的能力
- 为后续 RLHF 提供一个合理的起点

**本项目 SFT 实现（`1-SFT.py`）：**
```python
# 核心：用电商评论数据做 causal LM 训练
# input: "这本书真的很好" → target: "书真的很好看"
loss = F.cross_entropy(shift_logits, shift_labels, ignore_index=-100)
```

### 1.4 为什么 SFT 后还需要 RLHF？

SFT 的本质是 **模仿学习**（behavior cloning）：
- 模型学会了"像训练数据一样回复"
- 但训练数据中"好回复"和"差回复"混杂
- 模型无法区分哪个回复更好

**RLHF 解决的问题：**
- 用 Reward Model 学习"什么是好回复"
- 用 PPO 优化策略，使模型倾向于生成高奖励回复
- 通过 KL 约束防止模型偏离太远（保持语言质量）

### 1.5 PPO 在整个流程中的位置

```
                    ┌──────────────────────────────┐
                    │      PPO Optimization         │
                    │                              │
  Prompt ──→ Policy Model ──→ Response ──→ Reward Model ──→ Score
                    ↑                              │
                    │         KL Penalty           │
                    │              ↑               │
                    │     Reference Model          │
                    └──────────────────────────────┘
```

---

## 2. PPO 核心思想

### 2.1 从 Policy Gradient 到 PPO 的演进

#### Policy Gradient（策略梯度）

**核心思想：** 直接对策略参数 θ 求梯度，使高奖励动作的概率增大。

```python
# 原始策略梯度
loss = -log_prob(action) * reward
```

**问题：**
- **高方差**：reward 的波动导致梯度估计噪声大
- **学习率敏感**：步长太大 → 灾难性更新；步长太小 → 收敛慢
- **样本效率低**：每批数据只能用一次

#### TRPO（Trust Region Policy Optimization）

**核心思想：** 用 KL 散度约束策略更新的幅度，确保新策略不会偏离旧策略太远。

$$\max_\theta \mathbb{E}\left[\frac{\pi_\theta(a|s)}{\pi_{\theta_{old}}(a|s)} A(s,a)\right] \quad \text{s.t.} \quad \mathbb{E}[KL(\pi_{\theta_{old}} || \pi_\theta)] \leq \delta$$

**问题：**
- 需要计算 KL 散度的约束（二阶优化）
- 计算复杂，工程实现困难
- 需要多次采样来估计 KL 约束

#### PPO（Proximal Policy Optimization）

**核心思想：** 用 **clipped surrogate objective** 简化 TRPO 的约束，实现一阶近似。

```python
# PPO 的核心：clip 机制
ratio = torch.exp(new_log_probs - old_log_probs)
pg_loss1 = -ratio * advantages
pg_loss2 = -torch.clamp(ratio, 1-eps, 1+eps) * advantages
loss = torch.mean(torch.max(pg_loss1, pg_loss2))
```

**PPO 的优势：**
- 只需一阶优化（不需要二阶近似）
- 同一批数据可以重复使用（多轮更新）
- 训练稳定，超参数不敏感

### 2.2 PPO 解决的核心问题

| 问题      | PPO 的解决方案                    |
| ------- | ---------------------------- |
| 更新步长不可控 | Clip 限制 ratio 范围在 [1-ε, 1+ε] |
| 样本效率低   | 同一批数据多轮更新（ppo_epochs）        |
| 训练不稳定   | Clipped surrogate objective  |

---

## 3. LLM 中的 PPO 数据流

### 3.1 一次完整 PPO Iteration 的数据流

```
┌─────────────────────────────────────────────────────────────────┐
│                    Phase 1: Rollout（生成）                       │
│                                                                 │
│  Prompt (query)                                                 │
│       ↓                                                         │
│  Policy Model (π_θ)  ──→  Response (token ids)                  │
│       ↓                                                         │
│  记录: old_log_probs (rollout 时的策略概率)                        │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                    Phase 2: Reward（打分）                        │
│                                                                 │
│  Query + Response                                               │
│       ↓                                                         │
│  Reference Model (π_ref)  ──→  ref_log_probs                    │
│       ↓                                                         │
│  KL = old_log_probs - ref_log_probs                             │
│       ↓                                                         │
│  Reward Model  ──→  score                                       │
│       ↓                                                         │
│  rewards = -β * KL (+ score at end)                             │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                    Phase 3: Advantage（优势估计）                  │
│                                                                 │
│  Value Model (V_θ)  ──→  values                                 │
│       ↓                                                         │
│  GAE (γ=1.0, λ=0.95)  ──→  advantages                          │
│       ↓                                                         │
│  Whitening (归一化)                                              │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                    Phase 4: PPO Update（更新）                    │
│                                                                 │
│  for epoch in ppo_epochs:                                       │
│      for mini_batch in batch:                                   │
│          new_log_probs = π_θ(a|s)      ← 重新前向传播              │
│          ratio = exp(new - old)                                  │
│          loss = clip_loss + 0.1 * value_loss                    │
│          loss.backward() → optimizer.step()                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 每一步对应的代码函数

| 步骤 | 代码函数 | 文件 |
|------|----------|------|
| Prompt → Response | `model.generate()` | `3-PPO.py` L315-325 |
| 记录 old_log_probs | `compute_rewards()` | `3-PPO.py` L188-216 |
| RM 打分 | `reward_model()` | `3-PPO.py` L330-337 |
| KL 惩罚 | `compute_rewards()` | `3-PPO.py` L199-201 |
| GAE 计算 | `compute_advantage()` | `3-PPO.py` L236-255 |
| PPO 更新 | `compute_loss()` + `ppo_update()` | `3-PPO.py` L258-297 |

### 3.3 完整代码流程（3-PPO.py 主循环）

```python
# Phase 1: Rollout
for i, query in enumerate(query_tensors):
    # Policy Model 生成 response
    query_response = model.generate(input_ids=query.unsqueeze(0), ...)

    # Reward Model 打分
    score = reward_model(query_response.unsqueeze(0), ...)

# Phase 2: 计算奖励
logprobs, rewards, values, masks = compute_rewards(
    input_data, query_tensors, response_tensors, score_tensors
)

# Phase 3: 计算优势
advantages, returns = compute_advantage(rewards, values, masks)

# Phase 4: PPO 更新
ppo_update(input_data, logprobs, masks, advantages, returns)
```

---

## 4. PPO 核心数学公式

### 4.1 Policy Ratio（策略概率比）

**公式：**

$$r_t(\theta) = \frac{\pi_\theta(a_t | s_t)}{\pi_{\theta_{old}}(a_t | s_t)}$$

**代码实现：**

```python
# 3-PPO.py compute_loss()
ratio = torch.exp(logprobs - old_logprobs)
```

**为什么用 log 计算：**
- 数值稳定性：log 概率相减比概率相除更稳定
- `exp(log_a - log_b) = a / b`
- 避免概率值过小导致的浮点下溢

**ratio 的含义：**

| ratio 值 | 含义 |
|----------|------|
| ratio = 1 | 新策略与旧策略相同 |
| ratio > 1 | 新策略认为该动作概率更大 |
| ratio < 1 | 新策略认为该动作概率更小 |

### 4.2 PPO Clip Loss

**公式：**

$$L^{CLIP}(\theta) = \mathbb{E}_t \left[ \min\left( r_t(\theta) \hat{A}_t, \ \text{clip}(r_t(\theta), 1-\varepsilon, 1+\varepsilon) \hat{A}_t \right) \right]$$

**代码实现（`compute_loss()`）：**

```python
# 未裁剪的损失
pg_loss1 = -ratio * advantages
# 裁剪后的损失
pg_loss2 = -torch.clamp(ratio, 1 - 0.2, 1 + 0.2) * advantages
# 取两者中较大的（等价于 min 取负后取 max）
pg_loss = masked_mean(torch.max(pg_loss1, pg_loss2), masks)
```

**四种情况分析：**

| Advantage | ratio       | 裁剪前  | 裁剪后        | 效果          |
| --------- | ----------- | ---- | ---------- | ----------- |
| A > 0     | ratio > 1+ε | 增加概率 | **阻止继续增加** | 防止过度exploit |
| A > 0     | ratio < 1+ε | 增加概率 | 不变         | 正常更新        |
| A < 0     | ratio > 1-ε | 降低概率 | 不变         | 正常更新        |
| A < 0     | ratio < 1-ε | 降低概率 | **阻止继续降低** | 防止过度avoid   |

**为什么取 min：**
- 保守估计：取更小的期望回报
- 确保更新不会过于激进
- 这是 PPO 稳定性的核心

### 4.3 Advantage Estimation（优势估计）

**为什么不能直接使用 reward：**

- Reward 是瞬时反馈，不反映长期回报
- 不同情境下相同的 reward 含义不同
- 需要估计"这个动作比平均水平好多少"

**核心概念：**

| 符号 | 含义 | 公式 |
|------|------|------|
| Q(s,a) | 状态-动作价值 | 期望回报 |
| V(s) | 状态价值 | 期望回报（不考虑具体动作） |
| A(s,a) | 优势函数 | Q(s,a) - V(s) |

**代码实现：**

```python
# Value Model 输出 V(s)
logits, values = model(**input_data)  # values: (B, T)

# Advantage = 期望回报 - 基线
# GAE 估计的就是这个差值
advantages, returns = compute_advantage(rewards, values, masks)
```

### 4.4 GAE（Generalized Advantage Estimation）

**TD Error（时序差分误差）：**

$$\delta_t = r_t + \gamma V(s_{t+1}) - V(s_t)$$

**GAE 公式：**

$$\hat{A}_t^{GAE} = \sum_{l=0}^{\infty} (\gamma \lambda)^l \delta_{t+l}$$

**代码实现（`compute_advantage()`）：**

```python
gamma, lam = 1.0, 0.95

for t in reversed(range(seq_length)):
    # 下一个状态的 V(s)
    nextvalues = values[:, t + 1] if t < seq_length - 1 else 0.0
    # TD 误差
    delta = rewards[:, t] + gamma * nextvalues - values[:, t]
    # GAE 递推
    lastgae = delta + gamma * lam * lastgae
    advantage_reversed.append(lastgae)
```

**γ（gamma）和 λ（lambda）的作用：**

| 参数 | 值 | 作用 |
|------|-----|------|
| γ | 1.0 | 折扣因子（本项目设为 1.0，因为 RM 打分是序列级的） |
| λ | 0.95 | 偏差-方差权衡：λ→0 低方差高偏差，λ→1 低偏差高方差 |

**为什么 LLM PPO 中大量使用 GAE：**
- LLM 的序列很长，MC 方差太大
- GAE 通过 λ 平滑 TD(0) 和 MC
- 提供更稳定的 advantage 估计

### 4.5 Value Loss

**公式：**

$$L^V = \mathbb{E}_t \left[ (V_\theta(s_t) - R_t)^2 \right]$$

**代码实现：**

```python
v_loss = masked_mean((vpreds - returns) ** 2, masks)
# 总损失 = policy loss + 0.1 * value loss
loss = pg_loss + v_loss * 0.1
```

**为什么 value loss 系数是 0.1：**
- 防止 value 更新过大干扰 policy
- Value function 是辅助目标，policy 才是核心
- 经验值：太大 → policy 不稳定；太小 → advantage 估计不准

---

## 5. KL Penalty 详解

### 5.1 为什么需要 Reference Model？

**没有 Reference Model 会发生什么 —— Reward Hacking：**

模型会学会"骗 RM"而非"真正变好"：
- 生成高分但不自然的文本（如重复"好评好评好评"）
- 找到 RM 的漏洞，生成 RM 误判为高分的内容
- 语言质量严重退化

**Reference Model 的作用：**
- 作为"锚点"，约束策略不要偏离太远
- KL 惩罚确保生成的文本仍然自然、流畅
- 平衡"获取高奖励"和"保持语言质量"

### 5.2 KL 惩罚的计算

**公式：**

$$KL(\pi_\theta || \pi_{ref}) = \log \pi_\theta(a|s) - \log \pi_{ref}(a|s)$$

**代码实现（`compute_rewards()`）：**

```python
# 获取两个模型的 log 概率
logp = F.log_softmax(logits[:, :-1, :], dim=-1)
ref_logp = F.log_softmax(ref_logits[:, :-1, :], dim=-1)

# 取每个位置实际 token 的 log 概率
labels = input_data["input_ids"][:, 1:]
logp = torch.gather(logp, 2, labels.unsqueeze(-1)).squeeze(-1)
ref_logp = torch.gather(ref_logp, 2, labels.unsqueeze(-1)).squeeze(-1)

# KL = log π_policy - log π_reference
kl = logp - ref_logp

# Token 级奖励 = -β * KL
beta = 0.2
rewards = -kl * beta
```

### 5.3 为什么逐 token 计算 KL？

- 每个 token 的生成都受 KL 约束
- 防止模型在序列中间"放飞自我"
- 与 RM 分数在最后一个 token 叠加，形成完整的奖励信号

### 5.4 β 的作用

| β 值 | 效果 |
|------|------|
| β → 0 | 几乎没有 KL 约束，容易 reward hacking |
| β → ∞ | 强约束，模型几乎不更新，保持 SFT 行为 |
| β = 0.2（本项目） | 经验值，平衡探索和稳定 |

---

## 6. Actor-Critic 结构

### 6.1 为什么 PPO 需要 Critic？

**只用 Reward 的问题：**
- Reward 是稀疏的（只有序列末尾有分数）
- 无法指导"每个 token 应该怎么生成"
- 方差大，训练不稳定

**Critic 的作用：**
- 估计每个状态的价值 V(s)
- 提供基线（baseline），减小 advantage 方差
- 使 reward 信号可以反向传播到每个 token

### 6.2 Actor-Critic 的两种实现

| 实现方式 | 代码文件 | 特点 |
|----------|----------|------|
| 共享 backbone | `3-PPO.py` ActorCriticModel | 参数效率高，工业标准 |
| 独立 backbone | `4-PPO-Revised.py` PolicyNet + ValueNet | 结构清晰，教学用 |

**本项目的 ActorCriticModel（`3-PPO.py`）：**

```python
class ActorCriticModel(nn.Module):
    def __init__(self, model_path):
        self.llm = GPTModel(BASE_CONFIG)      # 共享 backbone
        self.v_head = nn.Linear(emb_dim, 1)    # Value head

    def forward(self, input_ids):
        outputs = self.llm(input_ids)
        lm_logits = outputs["logits"]              # Actor 输出
        value = self.v_head(outputs["last_hidden_state"]).squeeze(-1)  # Critic 输出
        return lm_logits, value
```

### 6.3 REINFORCE vs Actor-Critic vs PPO

| 特性 | REINFORCE | Actor-Critic | PPO |
|------|-----------|--------------|-----|
| 基线 | 无 | V(s) | V(s) + Clip |
| 更新方式 | 每次数据用一次 | 每次数据用一次 | 同一批数据多轮更新 |
| 方差 | 高 | 中 | 低 |
| 稳定性 | 差 | 中 | 好 |
| 样本效率 | 低 | 中 | 高 |

---

## 7. 代码实现解析

### 7.1 Rollout 阶段

**对应代码：`3-PPO.py` L302-339**

```python
for i, query in enumerate(query_tensors):
    query = torch.tensor(query, dtype=torch.long, device=device)

    # 随机决定生成长度
    new_tokens = random.choice(list(range(output_min_length, output_max_length)))

    # Policy Model 生成 response
    query_response = model.generate(
        input_ids=query.unsqueeze(0),
        max_new_tokens=new_tokens,
    ).squeeze(0)

    # 拆分出 response
    response_len = len(query_response) - len(query)
    response_tensors.append(query_response[-response_len:])
    query_response_tensors.append(query_response)

    # Reward Model 打分
    with torch.no_grad():
        query_response_score = torch.cat([
            query_response,
            torch.tensor([REWARD_TOKEN_ID], device=device),
        ])
        score = reward_model(query_response_score.unsqueeze(0), ...).squeeze(0)[-1]
        score = 2 * (score - 0.5)  # [0,1] → [-1,1]

    score_tensors.append(score)
```

**保存的数据：**
- `query_response_tensors`: 完整的 query + response
- `response_tensors`: 仅 response 部分
- `score_tensors`: RM 打分

### 7.2 Reward 计算

**对应代码：`compute_rewards()` L188-216**

```python
def compute_rewards(input_data, query_tensors, response_tensors, score_tensors):
    with torch.no_grad():
        # 1. 获取两个模型的 logits
        logits, values = model(**input_data)
        ref_logits, _ = ref_model(**input_data)

        # 2. 计算 log 概率
        logp = F.log_softmax(logits[:, :-1, :], dim=-1)
        ref_logp = F.log_softmax(ref_logits[:, :-1, :], dim=-1)

        # 3. 提取每个位置实际 token 的 log 概率
        labels = input_data["input_ids"][:, 1:]
        logp = torch.gather(logp, 2, labels.unsqueeze(-1)).squeeze(-1)
        ref_logp = torch.gather(ref_logp, 2, labels.unsqueeze(-1)).squeeze(-1)

        # 4. KL 惩罚
        kl = logp - ref_logp
        beta = 0.2
        rewards = -kl * beta

        # 5. 在最后一个 response token 加上 RM 分数
        for j in range(len(query_tensors)):
            end = len(query_tensors[j]) - 1 + len(response_tensors[j])
            rewards[j, end - 1] += score_tensors[j]

    return logp, rewards, values[:, :-1], masks
```

### 7.3 Advantage 计算

**对应代码：`compute_advantage()` L236-255**

```python
def compute_advantage(rewards, values, masks):
    lastgae = 0.0
    advantage_reversed = []
    gamma, lam = 1.0, 0.95

    # 反向遍历（GAE 需要从后往前递推）
    for t in reversed(range(seq_length)):
        nextvalues = values[:, t + 1] if t < seq_length - 1 else 0.0
        # TD 误差
        delta = rewards[:, t] + gamma * nextvalues - values[:, t]
        # GAE 递推
        lastgae = delta + gamma * lam * lastgae
        advantage_reversed.append(lastgae)

    # 反转回来
    advantages = torch.stack(advantage_reversed[::-1], dim=1)

    # Whitening 归一化
    advantages = masked_whiten(advantages, masks)

    # 计算折扣回报
    returns = advantages + values

    return advantages, returns
```

### 7.4 PPO Update

**对应代码：`ppo_update()` L268-297**

```python
def ppo_update(input_data, logprobs, masks, advantages, returns):
    for ep in range(ppo_epochs):  # 多轮更新
        for start in range(0, current_batch_size, mini_batch_size):
            # 取 mini-batch
            mini_batch_inds = batch_inds[start:start + mini_batch_size]

            # 重新计算当前策略的 log 概率（模型已更新）
            mb_logits, mb_vpreds = model(**mb_model_inputs)
            mb_logprobs = torch.gather(...)

            # 计算 PPO loss
            loss = compute_loss(
                logprobs[mini_batch_inds],  # 旧策略（rollout 时记录）
                mb_logprobs,                # 当前策略
                mb_vpreds[:, :-1],
                masks[mini_batch_inds],
                advantages[mini_batch_inds],
                returns[mini_batch_inds],
            )

            optimizer.zero_grad()
            loss.backward()
            optimizer.step()
```

---

## 8. 面试高频问题

### 基础概念

**Q1: PPO 是什么？**
> PPO（Proximal Policy Optimization）是 OpenAI 提出的策略梯度算法，通过 clipped surrogate objective 限制策略更新幅度，实现稳定训练。核心思想是：用新旧策略的概率比 ratio 衡量更新幅度，通过 clip 确保 ratio 在 [1-ε, 1+ε] 范围内。

**Q2: PPO 为什么适合 RLHF？**
> 1. 样本效率高：同一批数据可以多轮更新（ppo_epochs）
> 2. 训练稳定：clip 机制防止灾难性更新
> 3. 实现简单：只需一阶优化（相比 TRPO 的二阶优化）
> 4. 超参数鲁棒：clip 范围 0.2 在大多数场景都有效

**Q3: RLHF 中的四个模型分别是什么？**
> 1. **Policy Model**：当前正在优化的策略 π_θ
> 2. **Reference Model**：SFT 后冻结的模型 π_ref，用于 KL 约束
> 3. **Reward Model**：评估回复质量的打分模型
> 4. **Value Model**（Critic）：估计状态价值 V(s)，计算 advantage

**Q4: 为什么需要 Reference Model？**
> 防止 reward hacking。如果没有 reference model，模型会学会"骗 RM"——生成高分但不自然的文本（如重复"好评好评好评"）。KL 惩罚确保策略不要偏离 SFT 模型太远，保持语言质量。

### 数学原理

**Q5: ratio 是什么意思？**
> `ratio = π_new(a|s) / π_old(a|s)`，衡量新策略相比旧策略概率变化的倍数。ratio > 1 表示新策略更倾向于该动作，ratio < 1 表示更不倾向于。PPO 通过 clip 限制 ratio 在 [0.8, 1.2] 范围内。

**Q6: 为什么用 log 计算 ratio？**
> 数值稳定性。概率值可能非常小（如 1e-10），直接相除会浮点下溢。用 log 相减再 exp：`exp(log_prob_new - log_prob_old) = prob_new / prob_old`。

**Q7: advantage 怎么算？**
> `A(s,a) = Q(s,a) - V(s)`，其中 Q 是状态-动作价值，V 是状态价值。GAE 通过递推估计：`Â_t = δ_t + γλ * Â_{t+1}`，其中 `δ_t = r_t + γV(s_{t+1}) - V(s_t)` 是 TD 误差。

**Q8: GAE 为什么有效？**
> 通过 λ 参数平衡偏差和方差。λ→0 退化为 TD(0)（低方差高偏差），λ→1 退化为 MC（低偏差高方差）。λ=0.95 是经验值，在两者之间取得平衡。LLM 序列很长，MC 方差太大，GAE 提供更稳定的估计。

**Q9: 为什么取 min(ratio*A, clip_ratio*A)？**
> 保守估计。当 A>0 时，min 会选择更小的值，阻止 ratio 过大；当 A<0 时，min 会选择更小的绝对值，阻止 ratio 过小。这确保了更新不会过于激进。

### 工程实践

**Q10: KL 过大怎么办？**
> 1. 减小 β（KL 惩罚系数）
> 2. 减小学习率
> 3. 增加 ppo_epochs（多轮小步更新）
> 4. 检查 RM 是否合理
> 5. 监控 KL 曲线，及时 early stopping

**Q11: PPO 训练不稳定怎么排查？**
> 1. 检查 loss 曲线是否剧烈震荡
> 2. 检查 KL 是否持续增大（reward hacking 信号）
> 3. 检查 advantage 是否正确 whitening
> 4. 检查 reward 分布是否合理
> 5. 减小学习率或 clip 范围

**Q12: 为什么 value loss 系数是 0.1？**
> 防止 value 更新过大干扰 policy。Policy 是核心目标，Value 是辅助。系数太大 → policy 不稳定；太小 → advantage 估计不准。0.1 是经验值。

**Q13: ppo_epochs 一般设多少？**
> 本项目设为 4，InstructGPT 论文用 4。太多 → 旧 logprobs 失效严重；太少 → 样本效率低。通常 3~5 轮。

### 对比分析

**Q14: PPO vs DPO？**
> | 特性 | PPO | DPO |
> |------|-----|-----|
> | 需要 RM | 是 | 否 |
> | 需要 Reference Model | 是 | 是 |
> | 训练复杂度 | 高（4个模型） | 低（2个模型） |
> | 效果上限 | 高 | 中 |
> | 工程难度 | 高 | 低 |
>
> DPO 直接从偏好数据学习，跳过了 RM 训练，但效果上限通常不如 PPO+RM。

**Q15: PPO vs GRPO？**
> GRPO（Group Relative Policy Optimization）是 DeepSeek 提出的：
> - 不需要 Value Model
> - 用组内相对排名计算 advantage
> - 本项目 `day04/GRPO.py` 实现了这个算法
> ```python
> # GRPO 的 advantage 计算
> advantages = [(r - mean_reward) / std_reward for r in rewards]
> ```

**Q16: PPO vs SFT？**
> | 特性 | SFT | PPO |
> |------|-----|-----|
> | 目标 | 模仿数据 | 最大化奖励 |
> | 损失 | Cross-entropy | Clipped surrogate |
> | 需要 RM | 否 | 是 |
> | 效果 | 依赖数据质量 | 可超越数据质量 |

### 深入问题

**Q17: 为什么 PPO 需要同时优化 policy 和 value？**
> Policy loss 需要 advantage，而 advantage 依赖 value function。如果 value 不准，advantage 就不准，policy 更新方向就会错。所以需要同时训练 value function。

**Q18: Reward Hacking 是什么？如何避免？**
> 模型学会"骗 RM"——生成高分但不自然的文本。避免方法：
> 1. KL 惩罚（本项目的做法）
> 2. RM ensemble（多个 RM 投票）
> 3. 定期 human evaluation
> 4. Reward model 的正则化

**Q19: 为什么 LLM PPO 中 γ 通常设为 1.0？**
> 因为 RM 打分是序列级的（只在最后一个 token 有分数），不折扣意味着"每个 token 都同等重要"。如果 γ<1，后面的 token 权重会衰减，但 LLM 生成中每个 token 都很关键。

**Q20: 为什么用 masked_whiten 而不是普通 whitening？**
> LLM 序列有 padding，普通 whitening 会把 padding 位置也算进去，导致归一化不正确。Masked whitening 只在有效 token 位置计算均值和方差。

**Q21: PPO 的 clip 范围 ε 一般设多少？**
> 本项目和 InstructGPT 论文都用 0.2。经验值：
> - ε 太小 → 更新太保守，收敛慢
> - ε 太大 → 更新太激进，不稳定
> - 0.2 在大多数场景都有效

**Q22: 为什么需要 Whitening（白化）？**
> 1. 不同 batch 的 advantage 量级不同，学习率不稳定
> 2. Whitening 标准化为均值 0、方差 1
> 3. 加速收敛，减少梯度爆炸/消失

**Q23: 如何监控 PPO 训练是否正常？**
> 关键指标：
> 1. **Policy loss**：应该缓慢下降，剧烈震荡说明不稳定
> 2. **Value loss**：应该缓慢下降
> 3. **KL divergence**：应该缓慢增大，突然飙升说明 reward hacking
> 4. **Mean reward**：应该缓慢上升
> 5. **Clip fraction**：ratio 被 clip 的比例，通常 0.05~0.2

**Q24: PPO 训练的 batch_size 和 mini_batch_size 怎么选？**
> - batch_size：GPU 显存允许的最大值（本项目 32）
> - mini_batch_size：通常 4~8（本项目 4）
> - mini_batch 越小 → 梯度噪声越大，但可能跳出局部最优
> - mini_batch 越大 → 梯度越准，但可能过拟合当前 batch

**Q25: 为什么 PPO 中 old_log_probs 需要 detach？**
> 因为 old_log_probs 是 rollout 时记录的，不应该参与当前的梯度计算。如果不断裂计算图，旧策略的概率会随新策略更新而变化，破坏 ratio 的含义。

---

## 9. PPO 项目面试介绍

### 3 分钟自我介绍（模板）

> **项目背景：**
> 我在这个项目中实现了一个完整的 InstructGPT/RLHF 流程，从零开始手写 GPT-2 模型，用 PPO 算法微调，目标是让模型生成人类偏好的回复。
>
> **技术方案：**
> 整个流程分四个阶段：预训练、SFT、奖励模型训练、PPO 优化。我从零实现了 GPT-2 的完整架构（Multi-Head Attention、Transformer Block、LM Head），不依赖 transformers 库。权重从 HuggingFace 预训练模型加载，通过逐层映射解决格式差异。
>
> **模型结构：**
> PPO 阶段使用 Actor-Critic 架构——一个 GPT-2 backbone 同时输出 policy logits 和 value。此外还有 Reference Model（SFT 模型的冻结副本）用于 KL 约束，以及 Reward Model（GPT-2 + reward head）用于打分。
>
> **训练流程：**
> 每次 PPO iteration 分四步：
> 1. Rollout：Policy Model 根据 prompt 生成 response
> 2. Reward：Reward Model 打分，加入 KL 惩罚
> 3. Advantage：用 GAE（λ=0.95）估计优势，whitening 归一化
> 4. Update：PPO clipped surrogate objective，4 轮 mini-batch 更新
>
> **关键技术：**
> - KL 惩罚逐 token 计算，防止 reward hacking
> - GAE 平衡偏差-方差，whiten 归一化稳定训练
> - PPO clip 限制 ratio 在 [0.8, 1.2]，确保更新稳定
> - 同一批数据多轮更新，提高样本效率
>
> **遇到的问题与优化：**
> 1. HuggingFace 权重格式与手写模型不匹配——通过逐层 np.split + 转置解决
> 2. RM 打分范围 [0,1] 不适合 PPO——通过 2*(score-0.5) 调整到 [-1,1]
> 3. Padding 位置干扰 loss 计算——通过 mask 和 ignore_index=-100 解决

---

## 10. 核心公式速查表

| 公式 | 含义 | 代码位置 |
|------|------|----------|
| `ratio = exp(new_log_probs - old_log_probs)` | 策略概率比 | `compute_loss()` |
| `pg_loss = -min(ratio*A, clip(ratio)*A)` | PPO clipped loss | `compute_loss()` |
| `v_loss = (V(s) - returns)²` | Value loss | `compute_loss()` |
| `loss = pg_loss + 0.1 * v_loss` | 总损失 | `compute_loss()` |
| `δ_t = r_t + γV(s_{t+1}) - V(s_t)` | TD 误差 | `compute_advantage()` |
| `Â_t = δ_t + γλ * Â_{t+1}` | GAE 递推 | `compute_advantage()` |
| `KL = log π_policy - log π_reference` | KL 散度 | `compute_rewards()` |
| `reward = -β * KL + score` | Token 级奖励 | `compute_rewards()` |
| `whitened = (x - mean) / sqrt(var + eps) + mean` | Whitening | `masked_whiten()` |

---

## 11. 知识点关联图

```
                        ┌──────────────┐
                        │  RLHF 流程    │
                        └──────┬───────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
   ┌────▼────┐           ┌────▼────┐           ┌────▼────┐
   │   SFT   │           │   RM    │           │   PPO   │
   └────┬────┘           └────┬────┘           └────┬────┘
        │                      │                      │
        │                      │                      ├─→ Clipped Surrogate
        │                      │                      ├─→ GAE
        │                      │                      ├─→ KL Penalty
        │                      │                      └─→ Actor-Critic
        │                      │
        │                 Reward Hacking ──→ Reference Model
        │
   Behavior Cloning ──→ 无法区分好坏回复
```
