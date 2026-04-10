---
title: 'HyperAgents 论文学习'
date: 2026-04-08
permalink: /posts/2026/04/hyperagents/
tags:
  - HyperAgents
  - 超智能体
  - DGM
  - DGM-H
  - Darwin Gödel Machine
  - 达尔文戈德尔机
---

自主改进的人工智能（Self-improving AI）旨在通过学习改进自身的学习和问题解决过程，这种能力是未来通用人工智能（AGI）的核心能力之一。当前的递归自我改进方法通常依赖于固定的、手工设计的元级机制，这限制了此类系统的改进速度和“天花板”。

[HyperAgents](https://arxiv.org/abs/2603.19461) 是在达尔文哥德尔机（DGM）的基础上改进，相关背景概念如下：

## 背景概念

### 哥德尔机 (Gödel Machine)
*   **起源**：由 Jürgen Schmidhuber 提出，是一种理论上的**全自我指涉（Fully Self-referential）**、最优通用自我改进系统。
*   **核心逻辑**：它通过形式化证明（Formal Proofs）来重写自己的代码。只有当系统能证明某种修改能带来确定性的收益时，才会实施修改。
*   **局限性**：在现实世界的复杂任务中，几乎无法证明代码修改的长期收益，因此纯理论的哥德尔机在实践中极难实现。

### 达尔文哥德尔机 (Darwin Gödel Machine, DGM)
*   **演进**：DGM 是对哥德尔机的实证化改造。它将“形式化证明”替换为“实证验证（Empirical Validation）”。
*   **进化机制**：受生物进化启发，DGM 维护一个包含多个智能体变体的存档（Archive）。它不断从存档中采样并产生变异（修改代码），只有在基准测试中表现更好的子代才会被保留。
*   **局限性**：DGM 依赖一个手工设计的、固定的指令生成机制来指导自我改进。这导致它在编码以外的领域表现受限，因为它无法改进“如何改进”的逻辑。

HyperAgents 将任务逻辑和改进逻辑整合到单一的可编辑程序中（称为 `DGM-H`）。这意味着负责生成改进的代码（元智能体）本身也是可编辑的。系统不仅在学习解决任务，还在**学习“如何改进自己”的方法**，从而实现了元认知层面的进化。

## 2. HyperAgents 核心流程

![](/images/202604/hyperagents-1.png)

在 HyperAgents 系统中，存在两个核心的 Agent 角色：**Meta Agent（元智能体）** 和 **Task Agent（任务智能体）**。它们通过“生成-评估-优化”的进化循环（Evolutionary Loop）相互配合，实现系统的自我改进和任务求解能力的提升。

* **HyperAgents（超智能体）**：系统级的调度者与管理者。负责在全局维度协调和启动各个子 Agent 的进化循环，掌控整体的演化方向与迭代过程。
* **Task Agent（任务智能体）**：执行者。负责解决具体的底层领域任务（如编写代码、解决逻辑问题等）。它接收特定的问题描述（Problem Statement），并在给定的环境中（如 Git 仓库）尝试解决问题，输出预测结果或代码补丁。
* **Meta Agent（元智能体）**：进化引擎/优化者。负责改进 Task Agent 的代码。它扮演“程序员”的角色，能够查看过往 Task Agent 的评估结果和历史聊天记录，进而修改相关系统代码，使 Task Agent 变得更聪明。

这里的要点是： 初始时，Task Agent 和 Meta Agent 共享一个相同的 Agent 处理逻辑（`AgentSystem`），仅初始指令不同。 对 `AgentSystem` 的修改同时影响 Task Agent 和 Meta Agent 。

---

## 3. 进化循环 (The Generation Loop)

Meta Agent 和 Task Agent 嵌套在一个外层的进化评估循环中协同工作。整个配合过程是一个不断迭代的树形繁衍与反馈回路。

![](https://cdn-0.plantuml.com/plantuml/png/TLDTJnf157tVNp7H5ts0fDz9qnYjRNonZIacJLFwC8KHDYoxP9dLcgQ9LX14j3IYj6hzAA4QZM1ZQX04_ypExlBKlz0xEzL5q_fWW7lEkVUUSsSmIR5EZRWY9MEoci0wZgEG5azeAb5fW4uh1EaaHB4QKKWN9853CHmX_mSy8HFOKEWuUKU753cYna4EXU0VqIKfZ2b-YvC4zGn5-Ez3UYWgJvAWsYU-iwClrcwvlmVX91hMmiJ5zeufHboIH6UWFftK58FAcehrQJIgQGdKvrniiFmQQn_OopLWlvSa17-l0qQ2wAvo4beutmyxCmjc_SXUJzdPKxDPTaxJrjU6CzSoMolzRtZJCQm3MWpJEof7elInOX0XTpM0A8yGbUYOpy4RSe3RyncX6Ba8QCdISpSQy7At_8XEYEgnnt4oTiKscqsMsxqXm17T0W6in3MXy47CwPayJf8K1OYxFIkaxBqbwzUYKrljRCmBZpc0r_zglIAAeasv8bYdyWGEKMutLWntJ_I293HTU5bBcUV7Lgx2FkU458U9EgYzZCgKtD3XUOA_D1HrLolRExFERiMiptfnkzLBjeZmEPb2Nep0jDQExU8syDbHWLMtpFQYy6KuAYlXRcF6P3KcHBf3zVi7kZ9uY5W-pGg7_u9vJdim5-VHVGFUs02wvqSiEszj5OGlGRMJMRAsKcPzsRoeMiKp-wHfDxV5waBAQ-43hSiqGEesI6tki-q6oyoBpCnwZXr_Wm_Mpu_ENfdJFGAGFVyv_uuVEPa3bjk7Pm5PWKHG1TkiR86rIp0ap6ILAkoYn7MkSNs-wmRTzIEpjMdM6-pJKkT7rccli7Gbg5hv3AkTc_L5ipxZjAhY0OgbF4jyEWy21BrmGF4-xDOesiLz4NzG5IhWYxXy2Uxuwl7pSRnWEFsM7rdv1NlXZ5NNxMh9EbdzVPw7Xyqo3N4F7lcHptUzzOFBPVWLhQVWXFWXvNPW9McGg64Zhlm1)

一个完整的进化迭代周期（Generation Loop）包含以下关键步骤：

1. **父代实例选择**：
   从系统维护的 **Agent 存档（Codebase/实例库）** 中，基于特定算法选出一个表现较好的实例作为父代（Parent Agent）。
2. **在基准测试上尝试求解 (Task Agent 工作)**：
   系统唤醒父代中的 **Task Agent**，针对大量的测试集（如 Polyglot 编程基准测试）并发运行，尝试解决具体的问题。
3. **自动化评测**：
   系统对 Task Agent 的所有解答进行自动化评测，生成评估报告（Eval Reports，包含得分、失败日志等），并进行存档。
4. **作为反馈输入**：
   这些评估结果将作为反馈，喂给父代实例中的 **Meta Agent**。
5. **重构与修改系统代码 (Meta Agent 工作)**：
   **Meta Agent** 扮演开发者的角色，深入分析上述“错题本”，通过修改底层系统逻辑、Prompt 或工具来解决失败案例，生成代表系统改进的 **代码补丁（Code Patch）**。
6. **应用补丁孵化下一代**：
   将代码补丁应用到系统中，从而孵化出具有更强能力的新生代子实例（Child Agent）。
7. **作为新的子代加入**：
   新生成的子实例被重新加入到 **Agent 存档（Codebase）** 中，形成一棵不断生长、自我优化的进化树。

## 实验设置与结果

DGM-H 从一个简单初始 Agent 开发：该 Agent 配备了两个工具：一个用于执行shell命令的bash工具，以及一个用于检查和修改文件的专用工具。

DGM-H 在 `4` 个不同的领域（即编码、论文评审、机器人奖励设计和奥赛数学评分）进行测试与评估。为了降低计算成本，对于每个领域，首先在训练任务的一个小子集上评估智能体，以估计整体有效性。只有表现出足够性能的智能体才会在剩余的训练任务上进行评估。

最终结果如下（论文评审与奥赛数学评分）：

![](/images/202604/hyperagents-2.png)

更多结果可阅读原论文。
