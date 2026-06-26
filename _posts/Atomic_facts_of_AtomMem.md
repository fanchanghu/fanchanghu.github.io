# 🗃️ AtomMem 原子事实 (Atomic Facts)

**原始论文**: [AtomMem: Building Simple and Effective Memory System for LLM Agents via Atomic Facts](https://arxiv.org/abs/2606.19847)

### 🎯 核心定义
**原子事实**是记忆系统的“最小语义单元”。它通过专用提取器，将依赖上下文的原始对话转化为**自包含 (Self-contained)** 且**高信息密度**的结构化陈述。

---

### 📝 典型转换案例 (Case Study)
*假设当前对话时间元数据为：2023年5月17日*

*   **原始对话**: "Hey Zoe, guess what! **I** got an A on **my** first psychology exam **last Friday**! How's your week been?"
*   **原子事实**: **`["Emma got an A on her first psychology exam on the Friday before May 17, 2023."]`**
    *   *解析：代词“I/my”→“Emma”；相对时间“last Friday”→绝对日期；剔除社交废话。*

---

### 🧩 1. 结构化表示 ($F$)
每个原子事实存储为七元组：
- **$c$ (Content)**: 转换后的自包含文本。
- **$v$ (Embedding)**: 用于语义检索的密集向量。
- **$P$ (Participants)**: 涉及人员（如：Emma, Zoe）。
- **$K$ (Keywords)**: 核心关键词（如：心理学, 考试）。
- **$T$ (Temporal)**: 绝对时间戳或时间区间。
- **$E$ (Event ID)**: 所属“事件块”索引，用于关联更广泛的背景。
- **$id$**: 唯一标识符。

---

### ⚙️ 2. 抽取流程 (Extraction)
通过 **SFT (监督微调)** 训练的事实执行器执行：
1.  **去噪 (Denoising)**: 过滤掉问候、填充词等低价值对话。
2.  **共指消解 (Coreference)**: 将所有代词替换为明确实体。
3.  **时间锚定 (Anchoring)**: 将相对时间转化为可追溯的绝对时间。
4.  **第三人称重写**: 统一转换为客观陈述视角。

---

### 🛡️ 3. 存储与验证 (Verification)
-   **混合检索**: 基于向量 ($v$) + 关键词 ($K$) 找回最相关的旧记忆。
-   **残差存储**: 仅保存“非冗余”信息；若发现逻辑冲突，则生成更新指令。
-   **优势**: 这种“原子化”存储比存储原始对话更节省 Token，且能显著减少 LLM 幻觉。
