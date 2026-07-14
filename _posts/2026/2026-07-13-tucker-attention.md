---
title: '论文总结：Tucker Attention'
date: 2026-07-13
permalink: /posts/2026/07/tucker-attention/
tags:
  - Tucker Attention
  - Attention
  - Tensor Reformulation
  - 低秩分解
---

# 论文总结：Tucker Attention

## 一、研究动机与问题

Transformer 架构中的多头注意力（MHA）虽然强大，但引入了大量参数，导致三个阶段的内存瓶颈：

1. **训练阶段**：优化器状态和梯度占用大量显存
2. **推理阶段**：KV-cache 大小随头数和头维度线性增长
3. **注意力计算阶段**：Flash-Attention 等 kernel 中的 GPU 缓存压力

现有方法如 Group-Query Attention（GQA）和 Multi-Head Latent Attention（MLA）通过低秩分解来缓解这些问题，但作者指出这些方法存在一个根本性问题——它们都**在矩阵层面操作**，逐个处理 query、key、value、output 矩阵，无法充分利用跨头（cross-head）的冗余结构。

> "Instead of inspecting and manipulating the query, key, value and output matrices individually, we analyze the pre-softmax attention weights (query and key) and the post-softmax attention weights (value and output) as standalone objects."

## 二、张量重构视角（Tensor Reformulation）

### 2.1 融合权重的定义

对于每个头 $i$，定义两个融合后的权重对象。**Pre-softmax 权重**由 query 和 key 矩阵的乘积构成：

$$W_i := W^Q_i W^{K,\top}_i \in \mathbb{R}^{d_{\text{model}} \times d_{\text{model}}}$$

**Post-softmax 权重**由 value 和 output 矩阵的乘积构成：

$$\tilde{W}_i := W^V_i W^O_i \in \mathbb{R}^{d_{\text{model}} \times d_{\text{model}}}$$

其中 $W^Q_i, W^K_i, W^V_i \in \mathbb{R}^{d_{\text{model}} \times d_H}$，$W^O_i \in \mathbb{R}^{d_H \times d_{\text{model}}}$，$d_H = d_{\text{model}} / n_H$。

### 2.2 堆叠为三阶张量

将 $n_H$ 个头的融合权重按头索引堆叠，得到两个三阶张量：

$$\mathcal{W} = (W_i)_{i=1}^{n_H} \in \mathbb{R}^{n_H \times d_{\text{model}} \times d_{\text{model}}}$$

$$\tilde{\mathcal{W}} = (\tilde{W}^\top_i)_{i=1}^{n_H} \in \mathbb{R}^{n_H \times d_{\text{model}} \times d_{\text{model}}}$$

其中 $\mathcal{W}$ 的三个模式分别对应 **头模式**（head）、**query 模式**、**key 模式**；$\tilde{\mathcal{W}}$ 的三个模式分别对应 **头模式**（head）、**output 模式**、**value 模式**。

### 2.3 张量形式的 MHA 计算

定义中间张量：

$$\mathcal{H}^{(1)}(X) := \sigma\left(\frac{\mathcal{W} \times_2 X \times_3 X}{\sqrt{d_H}}\right) \in \mathbb{R}^{n_H \times N \times N}$$

$$\mathcal{H}^{(2)}(X) := \tilde{\mathcal{W}} \times_3 X \in \mathbb{R}^{n_H \times d_{\text{model}} \times N}$$

这里 $\times_j$ 表示 $j$-模式乘积（mode-$j$ product），$\sigma$ 是逐行 softmax 函数。$\mathcal{H}^{(1)}$ 收集了所有头产生的 $N \times N$ 注意力矩阵，$\mathcal{H}^{(2)}$ 收集了投影后的 value 表示。最终 MHA 输出通过**在头维度和序列维度上收缩**得到：

$$\text{MHA}(X; \mathcal{W}, \tilde{\mathcal{W}})_{jk} = \sum_{i,\ell} \mathcal{H}^{(1)}_{ij\ell}(X; \mathcal{W}) \mathcal{H}^{(2)}_{ik\ell}(X; \tilde{\mathcal{W}})$$

## 三、Tucker Attention 方法

### 3.1 Tucker 分解参数化

一旦将注意力权重表达为张量 $\mathcal{W}$ 和 $\tilde{\mathcal{W}}$，就可以引入 Tucker 分解——一种经典的低秩张量分解方法——来**同时压缩所有模式**。

**定义（Tucker Attention）**：对 pre-softmax 张量 $\mathcal{W} \in \mathbb{R}^{n_H \times d_{\text{model}} \times d_{\text{model}}}$ 和 post-softmax 张量 $\tilde{\mathcal{W}} \in \mathbb{R}^{n_H \times d_{\text{model}} \times d_{\text{model}}}$，其 Tucker 分解参数化为：

$$\mathcal{W} = \mathcal{C} \times_{j=1}^3 U_j, \quad \tilde{\mathcal{W}} = \tilde{\mathcal{C}} \times_{j=1}^3 \tilde{U}_j$$

其中：
- $\mathcal{C} \in \mathbb{R}^{r_1 \times r_2 \times r_3}$ 和 $\tilde{\mathcal{C}} \in \mathbb{R}^{\tilde{r}_1 \times \tilde{r}_2 \times \tilde{r}_3}$ 是**可学习的核心张量**
- $U_1 \in \mathbb{R}^{n_H \times r_1}$、$U_2 \in \mathbb{R}^{d_{\text{model}} \times r_2}$、$U_3 \in \mathbb{R}^{d_{\text{model}} \times r_3}$ 是**可学习的基矩阵**（$\tilde{U}_{1,2,3}$ 同理）
- Tucker 秩 $(r_1, r_2, r_3)$ 和 $(\tilde{r}_1, \tilde{r}_2, \tilde{r}_3)$ 分别控制头模式、query/output 模式、key/value 模式上的压缩程度

**核心洞察**：基矩阵 $U_2, U_3$（及 $\tilde{U}_2, \tilde{U}_3$）捕获了不同头之间在 query/key（output/value）计算中的**共享子空间**，而 $U_1$ 和 $\tilde{U}_1$ 则首次实现了**头模式本身的压缩**——这是 MHA、GQA、MLA 都做不到的。

### 3.2 参数效率分析

假设 Tucker Attention 中 key、query、value、output 的秩均设为 $r$，头模式秩设为 $r_1$，则参数数量为：

$$\text{Params}_{\text{Tucker}} = 2(n_H r_1 + 2r d_{\text{model}} + r_1 r^2)$$

相比之下，MHA 的参数数为 $4d_{\text{model}}^2$，GQA 为 $2d_{\text{model}}^2 + 2n_{KV} d_H d_{\text{model}}$，MLA 为 $d_{\text{model}}^2 + 5d_{\text{model}} d_c$。

当 $r \ll d_{\text{model}}$ 时，Tucker Attention 的参数规模随 $O(r d_{\text{model}} + r_1 r^2)$ 增长，而非 $O(d_{\text{model}}^2)$，因此具有显著的参数效率优势。

## 四、与现有方法的统一关系

一个关键的理论结果是：**MHA、MQA、GQA、MLA 都是 Tucker Attention 在特定秩下的特例**。

### 4.1 理论基础

**定理 B.2**：若 $W_i = W^Q_i W^{K,\top}_i$，则 pre-softmax 张量 $\mathcal{W}$ 可写为：

$$\mathcal{W} = \mathcal{C} \times_2 W^Q \Pi \times_3 W^K \Pi$$

其中 $\Pi$ 是列置换矩阵。因此，$W^Q$ 和 $W^K$ 的低秩分解直接决定了 $\mathcal{W}$ 的 Tucker 秩。

### 4.2 各方法的 Tucker 秩

| 方法 | $\mathcal{W}$ 的 Tucker 秩 | $\tilde{\mathcal{W}}$ 的 Tucker 秩 |
|------|--------------------------|----------------------------------|
| **MHA** | $(n_H, d_{\text{model}}, d_{\text{model}})$ | $(n_H, d_{\text{model}}, d_{\text{model}})$ |
| **MQA** | $(n_H, d_{\text{model}}, d_H)$ | $(n_H, d_{\text{model}}, d_H)$ |
| **GQA** ($n_{KV}$ 个 KV 头) | $(n_H, d_{\text{model}}, d_H n_{KV})$ | $(n_H, d_{\text{model}}, d_H n_{KV})$ |
| **MLA** | $(n_H, d^Q_c, d^K_c)$ | $(n_H, d_{\text{model}}, d^K_c)$ |

可以看到，**所有现有方法的头模式秩都是 $n_H$（即未压缩）**——这正是 Tucker Attention 的改进空间。

## 五、关键技术兼容性

### 5.1 KV-Caching

Tucker Attention 的 KV-cache 实现非常简洁：只需缓存 $K = XU_3 \in \mathbb{R}^{N \times r_3}$ 和 $V = X\tilde{U}_3 \in \mathbb{R}^{N \times r_3}$，缓存大小为 $2Nr_3$。若使用共享 KV 投影，则进一步减至 $Nr_3$。对于最后一个 token $x \in \mathbb{R}^{d_{\text{model}}}$ 的注意力更新：

$$\sigma\left(\frac{\tilde{\mathcal{C}} \times_1 U_1 \times_2 xU_2 \times_3 K}{\sqrt{d_H}}\right) \times_3 V$$

### 5.2 Latent RoPE

标准 RoPE 作用于头维度 $d_H$，但 Tucker Attention 参数化的是 $W^Q_i W^{K,\top}_i$ 的融合乘积，而非单独的 $W^Q_i$ 和 $W^K_i$。作者提出的解决方案是将旋转从**头维度移至 latent key 维度**。

**定义（Latent RoPE）**：令 $\hat{Q} = \mathcal{C} \times_1 U_1 \times_2 (XU_2) \in \mathbb{R}^{n_H \times N \times r_3}$，$K = XU_3 \in \mathbb{R}^{N \times r_3}$，则位置感知的注意力计算为：

$$\hat{Q}_m \times_3 \left(K_n R(n, r_3) R(m, r_3)^\top\right) \in \mathbb{R}^{n_H}$$

其中 $R(\ell, r_3) \in \mathbb{R}^{r_3 \times r_3}$ 是 latent key 空间中的旋转矩阵。该形式满足相对位置性质：

$$\hat{Q}_m \times_3 \left(K_n R(n, r_3) R(m, r_3)^\top\right) = \left(\hat{Q}_m \times_3 R(m, r_3)^\top\right) \times_3 \left(K_n R(n, r_3)\right) = \left(\hat{Q}_m \times_3 R(m-n, r_3)^\top\right) \times_3 K_n$$

### 5.3 MLA 的 RoPE 简化

一个重要的推论是：**Latent RoPE 使 MLA 不再需要解耦 RoPE（decoupled RoPE）**。原始 MLA 实现 (DeepSeek-V2) 将 query 和 key 矩阵划分为"语义"和"旋转"两部分，增加了架构复杂度。而通过 Latent RoPE，MLA 可以直接计算：

$$X_m W^{DQ} W^{UQ}_i W^{UV}_i R(m, d_c) R(n, d_c)^\top W^{DKV,\top} X_n$$

这允许在推理时将 query 侧矩阵融合为单一投影 $W^{Q,\text{MLA}}_i := W^{DQ} W^{UQ}_i W^{UV}_i \in \mathbb{R}^{d_{\text{model}} \times d_c}$，消除了对分离路径的需求。

### 5.4 Flash-Attention 兼容性

Tucker Attention 可**同时获得 MQA 和 MLA 在 Flash-Attention 中的优势**：
- **MQA 的优势**：单个 KV 对为所有 query 头共享，只需加载一次
- **MLA 的优势**：KV chunk 编码在 $r_3$ 维 latent 空间中，比 $d_H$ 维的 chunk 占用更少 SRAM

实现上，只需预计算 $K = XU_3$、$V = X\tilde{U}_3$ 和每个头的 $Q_i = \mathcal{C} \times_1 U_1 \times_2 (XU_2)$，然后调用原生支持 GQA 的 Flash-Attention kernel（以 $n_{KV}=1$ 的 GQA 形式调用）。

## 六、实验验证

### 6.1 奇异值分析

对训练后的 GPT-2 模型进行后验奇异值分析，作者发现 $\mathcal{W}$ 和 $\tilde{\mathcal{W}}$ 在所有模式上（包括头模式）都表现出显著的**谱衰减**，表明在实际训练中确实存在可压缩的低秩结构。MHA 对所有模式表现出相似的谱衰减，但 GQA 和 MLA 无法利用头模式中的冗余——只有 Tucker Attention 的 $U_1$ 基矩阵能捕获这种结构。

### 6.2 主要实验结果

- **ViT 迁移学习**：在 ImageNet1k 上，Tucker Attention 在达到可比精度时，参数比 GQA 和 MLA 少**近一个数量级**
- **GPT-2 预训练**：Tucker 秩 $(8, 128, 64)$ 仅需 MHA 约 **18%** 的参数和 MLA 约 **39%** 的参数即可达到竞争性的验证指标
- **LLaMA3-1B 训练**：Tucker Attention 仅使用 GQA ($n_{KV}=8$) 约 **10–20%** 的参数，验证损失仅增加 1–3%，同时训练迭代时间减少最高 20%
