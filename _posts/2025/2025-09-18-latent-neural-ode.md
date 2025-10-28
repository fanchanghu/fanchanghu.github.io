---
title: 'Latent Neural ODE'
date: 2025-09-18
permalink: /posts/2025/09/latent-neural-ode/
tags:
  - Neural ODE
  - time-series model
  - ODE
---

Latent Neural ODE 是 [Neural ODE](https://arxiv.org/abs/1806.07366v5) 中提出的一种时间序列模型（time-series model）。传统时间序列模型中，事件是随时间连续的，但数据采集通常是离散的，并且采集间隔也不固定。这对传统神经网络来说存在数据利用困难，预测结果难以在时间线上扩展等问题。而连续性正是 Neural ODE 所天然具备的。

相关文章：
* [Neural ODE 与伴随灵敏度方法（Adjoint Sensitivity Method）](/posts/2025/08/blog-post-1)
* [Continuous Normalizing Flow（CNF）](/posts/2025/08/blog-post-4)

---

## Latent Neural ODE 网络架构

![Illustration Latent Neural ODE](/images/202509/latent-ode.png)
图1

### 训练过程

1. **编码阶段（Encode）**：
   - 使用**RNN编码器**（如GRU或LSTM）从训练样本的观测序列 ` x_t0, ..., x_tN ` 中提取隐状态 `z`。
   - 输出初始隐状态的后验分布参数 `μ, σ` ，其中 `φ` 为编码器参数：
     $$
     N(\mu, \sigma) = q_\phi (z_{t_0} | x_{t_0}, ..., x_{t_N})
     $$

2. **采样初始隐状态（Sample）**：
   - 利用重参数化方法，从上述后验分布中采样得到 `z_t0 = μ + σ * ε` ，其中 `ε ~ N(0, 1)` 。

3. **ODE 演化（Flow）**：
   - 使用神经网络 `f_θ` 参数化的ODE求解器（如 `ODESolve`）演化隐状态：
     $$
     z_{t_i} = \text{ODESolve}(z_{t_0}, f_\theta, t_0, t_i)
     $$

4. **解码阶段（Decode）**：
   - 将每个时间点的隐状态 `z_ti` 输入解码器 `p_ψ` ，重构观测数据：
     $$
     \hat{x}_{t_i} \sim p_\psi (x | z_{t_i}, \theta_x)
     $$

5. **损失函数**：
   - Latent ODE 的损失函数是标准的 **变分下界（ELBO）**，包含两个部分：
    $$
    \mathcal{L} = \underbrace{-\mathbb{E}_{q_\phi (z_{t_0})} \left[ \sum_{i=1}^N \log p_\psi (x_{t_i} | z_{t_i}) \right]}_{\text{重构误差}} + \underbrace{D_{\text{KL}}(q_\phi (z_{t_0} | x_{t_0}, ..., x_{t_N}) \| p(z_{t_0}))}_{\text{KL正则项}}
    $$
   - 基于以上损失函数，联合优化编码器参数 `φ`、演化网络参数 `θ`、解码器参数 `ψ` 。

### 生成过程

Latent ODE 的生成过程（解码过程）是**从隐空间到数据空间的映射**，具备**连续时间建模能力**，可以在任意时间点生成数据。

### 生成步骤如下：

1. **生成初始隐状态**：
   - 从先验采样（无历史观测状态）
     $$
     z_{t_0} \sim \mathcal{N}(0, I)
     $$
   - 从后验采样（从历史观测状态，预测后续状态）
     $$
     z_{t_0} \sim q_\phi (z_{t_0} | x_{t_0}, ..., x_{t_N})
     $$

2. **ODE 演化**：
   - 使用训练好的神经网络 `f_θ` 演化隐状态到任意目标时间 `t`：
     $$
     z_t = \text{ODESolve}(z_{t_0}, f_\theta, t_0, t)
     $$

3. **解码生成数据**：
   - 将 `z_t` 输入解码器，生成对应时间点的数据：
     $$
     \hat{x}_t \sim p_\psi (x | z_t, \theta_x)
     $$

由于ODE是连续时间的，该模型可以**在训练时未见过的时间点上进行外推（extrapolation）**，这是传统RNN难以实现的。

### 实验效果

![Illustration Latent Neural ODE](/images/202509/latent-ode-2.png)
图2

该图反应了 Latent ODE 在一维序列上的预测和外推能力。相比 RNN 有明显改善。
