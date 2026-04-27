# Transformer — Attention Is All You Need

## 元信息
- 已学习文档列表：
  - [x] `documents/Attention-Is-All-You-Need.md:530 lines`
- 知识领域：深度学习、自然语言处理、序列转录模型
- 学习日期：2026-04-27

---

## `documents/Attention-Is-All-You-Need.md`

### 背景与问题

> **速读总结**
> 序列转录（如机器翻译）长期以来以 RNN 或 CNN 为主力架构。RNN 的隐状态按时间步顺序计算，无法并行训练；CNN 虽然可并行，但远距离依赖的路径长度随距离增长（线性或对数）。两者都制约了长距离依赖的学习效率。
> 
> **可回答问题**：
> - 为什么 RNN 不适合长序列并行训练？
> - 为什么 CNN 在处理长距离依赖时有局限性？

### 3 Model Architecture — Transformer 整体架构

> **速读总结**
> Transformer 是第一个完全基于注意力机制的序列转录模型，弃用递推和卷积。沿用 encoder-decoder 结构，但两者都由堆叠的 self-attention 和 feed-forward 层组成。
> 
> **可回答问题**：
> - Transformer 的整体结构是怎样的？
> - Encoder 和 Decoder 各由什么组成？

#### 精读笔记

- **Encoder 结构**（`L72-L73`）：6 个相同层堆叠。每层有两个子层：(1) Multi-Head Self-Attention，(2) Position-wise Feed-Forward Network。每个子层采用残差连接 + Layer Normalization：`LayerNorm(x + Sublayer(x))`。所有子层输出维度 `d_model = 512`。

- **Decoder 结构**（`L76-L77`）：也是 6 个相同层堆叠。比 encoder 多一个子层：对 encoder 输出做 Multi-Head Attention。同时修改 self-attention 子层，通过 masking 防止当前位置关注后续位置，保证自回归属性。

- **证据/数据**：所有子层输出维度一致（`d_model = 512`），这是残差连接能正常工作的前提。

### 3.2 Attention — 注意力机制

> **速读总结**
> Attention 本质是将 query 和 key-value 对映射为输出——输出是 value 的加权和，权重由 query 和 key 的兼容性函数计算。Transformer 在此基础上提出了 Scaled Dot-Product Attention 和 Multi-Head Attention。
> 
> **可回答问题**：
> - Scaled Dot-Product Attention 的公式是什么？
> - 为什么要除以 √d_k？
> - Multi-Head Attention 解决了什么问题？

#### 精读笔记

- **Scaled Dot-Product Attention**（`L84-L93`）：
  - 公式：`Attention(Q,K,V) = softmax(QK^T / √d_k) V`
  - 相比 additive attention，dot-product attention 可用高度优化的矩阵乘法实现，更快、更省内存。
  - 缩放因子 `1/√d_k` 的关键作用：当 d_k 很大时，点积的值会变大，将 softmax 推入梯度极小的区域。数学原因：若 q、k 的分量是独立随机变量（均值 0，方差 1），点积均值为 0，方差为 d_k。

- **Multi-Head Attention**（`L100-L117`）：
  - 将 Q、K、V 分别做 h 次线性投影（不同、可学习的），并行计算 attention，拼接后再投影。
  - 作用：允许模型在不同位置联合关注来自不同表示子空间的信息。单头 attention 的均值化会抑制这一点。
  - 参数：h = 8，每个 head 的 d_k = d_v = d_model / h = 64
  - 关键 insight：每个 head 维度降低后，总计算量与单头全维度 attention 相近。

- **三种使用方式**（`L123-L128`）：
  1. Encoder-Decoder Attention：query 来自 decoder，key/value 来自 encoder 输出。每个 decoder 位置可以关注所有输入位置。
  2. Encoder Self-Attention：key、value、query 都来自上一层 encoder 输出。每个位置可以关注 encoder 前一层的所有位置。
  3. Decoder Self-Attention：每个位置只能关注到当前位置及之前的 decoder 位置（通过 mask 实现）。

### 3.3 Position-wise Feed-Forward Networks

> **速读总结**
> 每个 encoder/decoder 层除了 attention 子层，还有一个全连接前馈网络，对每个位置单独且相同地应用。
> 
> **可回答问题**：
> - FFN 的结构是什么？

#### 精读笔记
- **结构**（`L133-L137`）：两个线性变换 + ReLU 激活：`FFN(x) = max(0, xW₁ + b₁)W₂ + b₂`
- 等价于 kernel size 为 1 的两层卷积
- 输入输出维度 d_model = 512，中间层维度 d_ff = 2048

### 3.5 Positional Encoding — 位置编码

> **速读总结**
> Transformer 没有递推和卷积，无法天然感知序列顺序。因此将位置编码加到输入 embedding 上（直接在底部相加），让模型利用序列的顺序信息。
> 
> **可回答问题**：
> - 为什么 Transformer 需要位置编码？
> - 为什么选择正弦/余弦函数？

#### 精读笔记
- **公式**（`L149-L153`）：
  - PE(pos, 2i) = sin(pos / 10000^(2i/d_model))
  - PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
- **设计动机**：对于任意固定偏移 k，PE(pos+k) 可以表示为 PE(pos) 的线性函数，让模型容易学会关注相对位置。
- **实验对比**（`L158`）：可学习位置 embedding 与正弦版本结果几乎相同（见 Table 3 row E）。选择正弦版本是因为它可能外推到训练时未见过的序列长度。

### 4 Why Self-Attention — 为什么用 Self-Attention

> **速读总结**
> 从三个维度比较 self-attention、recurrent 和 convolutional 层：计算复杂度、可并行化程度、长距离依赖路径长度。
> 
> **可回答问题**：
> - Self-Attention 相比 RNN、CNN 的优势是什么？
> - Self-Attention 的局限性是什么？

#### 精读笔记
- **Table 1 核心数据**（`L171-L177`）：

| 层类型 | 每层复杂度 | 顺序操作数 | 最大路径长度 |
|:---|:---|:---|:---|
| Self-Attention | O(n²·d) | O(1) | O(1) |
| Recurrent | O(n·d²) | O(n) | O(n) |
| Convolutional | O(k·n·d²) | O(1) | O(log_k(n)) |

- **关键结论**：
  - Self-Attention 用常数次顺序操作连接所有位置，RNN 需要 O(n) 次。
  - 当序列长度 n < 表示维度 d 时（机器翻译中常见），Self-Attention 比 RNN 更快。
  - Self-Attention 的最大路径长度是 O(1)，远优于 RNN 的 O(n) 和 CNN 的 O(log_k(n))，更利于学习长距离依赖。
  - **局限性**：复杂度是序列长度的平方 O(n²·d)，对于极长序列开销很大。可通过限制关注邻域（restricted self-attention）缓解。

- **可解释性**：Self-Attention 还能提供更可解释的模型——不同 attention head 学会执行不同任务，有些与句子的句法和语义结构相关。

### 5 Training — 训练细节

> **速读总结**
> 在 WMT 2014 英德和英法数据集上训练，使用 Adam 优化器和热身学习率调度。采用了 Residual Dropout 和 Label Smoothing 正则化。

#### 精读笔记
- **优化器**（`L202-L208`）：Adam（β₁=0.9，β₂=0.98，ε=10⁻⁹）。学习率先线性升高（warmup_steps=4000），再按步数的平方根倒数衰减。
- **正则化**（`L214-L220`）：子层输出后 Residual Dropout（P_drop=0.1），Label Smoothing（ε_ls=0.1）。

### 6 Results — 实验结果

> **速读总结**
> Transformer 在英德和英法翻译上都达到了当时的 SOTA，训练成本仅为其他模型的一小部分。在英文成分句法分析上也表现优异。

- **英德翻译**（`L242`）：big model 28.4 BLEU，超过所有已有模型（含集成模型）2+ BLEU。训练 3.5 天 / 8 P100。
- **英法翻译**（`L244`）：big model 41.8 BLEU，训练成本不到之前 SOTA 模型的 1/4。
- **消融实验**（Table 3）：多头注意力优于单头；更大的模型效果更好；dropout 对防止过拟合很重要；正弦位置编码与可学习版本效果相当。
- **句法分析**（`L310-L312`）：无任务特调的情况下，Transformer 在 WSJ 仅 4 万句训练集上就超越了 BerkeleyParser。

### 三句话复述
1. **问题**：序列转录任务中，RNN 和 CNN 分别受限于顺序计算和长距离依赖路径，难以高效并行训练和捕捉全局依赖。
2. **方案**：提出 Transformer——完全基于多头自注意力机制的 encoder-decoder 架构，用位置编码注入顺序信息，用缩放点积注意力实现高效的全局依赖建模。
3. **结果**：在 WMT 2014 英德（28.4 BLEU）和英法（41.8 BLEU）翻译任务上达到新的 SOTA，训练成本大幅降低，并能泛化到句法分析等任务。
