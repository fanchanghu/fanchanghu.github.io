---
title: 'LLM Agent 检索式记忆学习'
date: 2026-01-30
permalink: /posts/2026/01/retrieval-based-memory/
tags:
  - 智能体记忆
  - Agent Memory
  - 检索式记忆
  - Retrieval-based Memory
---

## A-Mem

![A-Mem Archieve](/images/202601/rbm-1.png)

- **论文**： [A-Mem: Agentic Memory for LLM Agents](https://arxiv.org/abs/2502.12110)
- **一句话总结**： 参考 **卢曼卡片盒笔记法（Zettelkasten method）** 实现的一种动态、自我演化的记忆系统。
- **亮点**： 不仅存储和检索记忆内容，而且在记忆片段间 **建立链接、更新历史记忆**。
- **关键步骤**：
  - **笔记构建（Note Construction）**： 将原始交互内容转化为包含丰富元数据的结构化笔记。这些属性包括：上下文描述（Contextual Descriptions）、关键词（Keywords）、标签（Tags）、嵌入向量（用于语义匹配）等。
  - **链接生成（Link Generation）**： 系统会自动分析新笔记与已有历史记忆之间的关系。通过 LLM 的推理能力，在具有语义相关性或逻辑关联的笔记之间建立双向链接。这使得记忆从碎片化的列表演变为一个复杂的图结构网络。
  - **记忆进化（Memory Evolution）**： 这是 A-Mem 的独特之处。新记忆的加入不仅仅是增量，它还会触发对旧记忆的“反思”。系统会根据新信息更新现有笔记的上下文描述或标签，从而使记忆库能够随着时间的推移不断优化和深化理解。

> 与其说是“记忆”，不如说是“笔记”。—— 个人感觉：所有检索式记忆都更像“笔记”，参数化记忆和潜在记忆才是真正的“记忆”？

## H-MEM

![H-MEM Archieve](/images/202601/rbm-2.png)

* **论文**：[Hierarchical Memory for High-Efficiency Long-Term Reasoning in LLM Agents](https://arxiv.org/abs/2507.22925)
* **一句话总结**：一种类似图书馆分级索引系统的、自上而下、具备**路由检索**能力的记忆架构。
* **亮点**：通过**语义分层**解决**海量记忆**下的检索噪声与计算效率问题。
* **关键步骤**：
  * **分层组织 (Hierarchical Organization)**：将记忆按抽象程度划分为多个层级。底层存储原始交互细节，高层存储语义浓缩的摘要。这种非对称结构确保存储的内容既有广度（宏观）又有深度（微观）。
  * **位置索引编码 (Positional Index Encoding)**：在不同层级的记忆向量间植入“导航指针”。显式地定义了高层摘要与底层细节之间的归属关系，将记忆构建成具有层级路径的树状结构。
  * **路由检索机制 (Routing Mechanism)**：检索时不再扫描全库，而是从最高层开始匹配，通过 TopK 方式沿着“位置索引”逐层向下查找。过滤了无关背景带来的语义干扰。
  * **动态层级更新 (Hierarchical Update)**：论文说：“We also designed a self-adaptation hierarchy adjustment interface, allowing users to dynamically adjust the hierarchy structure based on the complexity of the current conversation and the semantic granularity of the memorized content.” 但并没有提到具体如何做 ...
  * **记忆更新 (Memory Update)**：根据用户的反馈表现（赞同、无明显反馈或反驳）来调整相应记忆的权重。上述增强或减弱过程通过与LLM生成的反馈权重相乘来进行更新，从而实现记忆强度的动态调整。

## G-Memory

![G-Memory Archieve](/images/202601/rbm-3.png)

* **论文**： [G-Memory: Self-Evolving Graph Memory for Multi-Agent Systems](https://arxiv.org/abs/2506.07398)
* **一句话总结**： 基于 **组织记忆理论（Organizational Memory Theory）** 构建的一种层级化、三层图结构的记忆系统，赋予 **多智能体** 团队在协作中“自我进化”的能力。
* **亮点**： 记忆包括 **洞察（宏观指导）**、**查询（问题描述）** 和 **交互（微观细节）** 三类数据，以及它们之间的关联关系，从而构成三个互相关联的图结构。
* **关键步骤**：
  * **粗粒度检索（Coarse-grained Memory Retrieval）**：
  当新任务到来时，系统利用语义搜索在“查询图”中定位相似案例，并沿着“查询图”中的边“跳跃扩展（hop expansion）”到更多查询节点。
  * **细粒度记忆合成**
    1. 使用 LLM 筛选最相关的 `M` 个查询
    2. 通过查询节点定位到对应的“交互图”节点，结合当前问题，提取核心子图，以过滤无关信息
    3. 通过查询节点定位到对应的“洞察图”节点
    4. 使用 LLM ，根据当前问题、步骤2的核心交互子图、步骤3的洞察节点、角色信息生成当前 Agent 的“个人洞察（personalized insights）”，作为该 Agent 的记忆
  * **记忆进化（Memory Evolution）**：
    任务执行完毕后，将基于当前任务产生新的记忆、或更新已有记忆的节点或边：
    1. 交互层面：G-Memory 追踪每个代理的发言，以构建交互图，然后将其存储。
    2. 查询层面：一个新的查询节点被实例化并添加到查询图。
       - 在当前查询和之前使用的前M个相关历史查询之间建立连接。
    3. 洞察层面：
       - 利用 LLM ，根据当前任务的交互轨迹，生成总结经验的可能的新洞察。
       - 利用 LLM ，更新之前使用的洞察，在新洞察和更新后的洞察之间建立连接。
       - 在当前查询和上述洞察（新洞察和更新后的洞察）之间建立连接。
