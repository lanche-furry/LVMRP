# IRIXEN / Patho-1 — LVMRP v2 架构改造工程笔记

# IRIXEN / Patho-1 — LVMRP v2 Architecture Rework: Engineering Notes

> **内部技术备忘 / Internal technical memo.** 不是交付文档，不是产品材料。语气直白，结论不加包装。所有数字直接引自实验日志，不做四舍五入改写。
>
> **Internal memo, not a deliverable.** Not product material. Plain tone, no hedging for presentation. Every number is quoted directly from the experiment log, unrounded.

---

## 1. 背景与目标 / Background and Goal

**中文**

ep25 两个指标同时难看：**SFS = 0.4452**，**条件敏感度 L1 = 0.0112**。

L1 = 0.0112 意味着改变条件输入，生成图几乎不动。诊断结果是：**全局 FiLM 是条件信号被平均化的结构性原因** —— 条件向量只在瓶颈 `e6` 注入一次，随后经过五层上采样，信号被稀释掉。

目标很直接：通过架构改造把条件敏感度拉起来，看 SFS 是否随之提升。

**English**

ep25 was bad on two numbers at once: **SFS = 0.4452** and **condition sensitivity L1 = 0.0112**.

L1 = 0.0112 means changing the conditioning input barely moves the generated image. The diagnosis: **the single global FiLM injection is the structural cause of the averaging.** The condition vector enters once, at bottleneck `e6`, and is then diluted across five upsampling layers.

The goal was simple: raise condition sensitivity architecturally and see whether SFS follows.

---

## 2. 四个组件 / Four Components

**本节是文档主体。** 每个组件写三件事：改了什么、为什么改、效果数字。

**This section is the body of the document.** Each component covers three things: what changed, why, and the resulting number.

### 2.1 组件 1：多层 FiLM / Component 1: Multi-layer FiLM

**中文**

- **改了什么。** 原版只在瓶颈 `e6` 注入一次：

  ```python
  self.film = nn.Linear(cond_dim, base * 8 * 2)
  ```

  改后 `d1`–`d5` 各有独立的 `film_d1`–`film_d5`，逐层注入。

- **为什么改。** 原版条件信号在 5 层上采样中被稀释。一次注入、五次上采样，等于把条件信号平均到接近常数。

- **效果。** **单独使用时 L1 从 0.0112 到 0.0949（8.5×）。**

**English**

- **What changed.** The original injected the condition exactly once, at the bottleneck:

  ```python
  self.film = nn.Linear(cond_dim, base * 8 * 2)
  ```

  Now `d1`–`d5` each own an independent `film_d1`–`film_d5`, injecting at every level.

- **Why.** The original condition signal was diluted across five upsampling steps. One injection followed by five upsamples averages the condition down to nearly a constant.

- **Effect.** **Used alone, L1 goes from 0.0112 to 0.0949 (8.5×).**

### 2.2 组件 2：数值约束 / Component 2: Numerical Constraints

**中文**

- **改了什么（三处）。**
  - `_film` 里 `g = torch.tanh(g)`、`b = torch.clamp(b, -2, 2)`；
  - `spatial_proj` 输出 `g = tanh(g)`、`bb = clamp(bb, -0.5, 0.5)`；
  - `cond_proj` 改为 `Sequential(Linear, LayerNorm, Tanh)`；
  - 所有 film 层小 std 初始化（**0.01**），**不继承 ep25 的 film**。

- **为什么改。** 不做约束时数值直接爆炸：`cond_std` 冲到 **173–418**，`d1` 到 **-14590**，最终 Tanh 饱和成纯白，**`fake_std = 0.00000`** —— 生成器完全塌缩，输出一张白图。

- **效果。** `cond_std` 压到 **0.63**，`fake_std` 到 **0.25–0.37**，生成器不再塌缩。

**English**

- **What changed (three places).**
  - in `_film`: `g = torch.tanh(g)`, `b = torch.clamp(b, -2, 2)`;
  - `spatial_proj` output: `g = tanh(g)`, `bb = clamp(bb, -0.5, 0.5)`;
  - `cond_proj` becomes `Sequential(Linear, LayerNorm, Tanh)`;
  - every film layer uses small-std init (**0.01**) and **inherits nothing from ep25's film**.

- **Why.** Without constraints the numbers blow up: `cond_std` reaches **173–418**, `d1` hits **-14590**, Tanh saturates to pure white, and **`fake_std = 0.00000`** — the generator collapses to a blank image.

- **Effect.** `cond_std` drops to **0.63**, `fake_std` sits at **0.25–0.37**, and the generator stops collapsing.

### 2.3 组件 3：d2 空间条件注入 / Component 3: Spatial Conditioning at d2

**中文**

- **改了什么。** tokens `[B, 256, 1536]` reshape 成 `[B, 1536, 16, 16]`，经 3 层卷积投影成 `[B, base*8*2, 16, 16]` 的 `(γ, β)` 图。注入点在 `d2`（16×16），**正好匹配 token 网格**。

- **为什么改。** 全局向量无法产生空间变化的调制。同一个条件向量在整幅图上给出同一个 γ/β，本质上仍是全局 FiLM，只是换了个位置注入。

- **效果。** 与组件 1、2 联合，**v2ep4 的 L1 = 0.0949（无标签监督时）**。

**English**

- **What changed.** Tokens `[B, 256, 1536]` are reshaped to `[B, 1536, 16, 16]` and projected through three convolutions into a `(γ, β)` map of shape `[B, base*8*2, 16, 16]`. Injection happens at `d2` (16×16), which **matches the token grid exactly**.

- **Why.** A global vector cannot produce spatially varying modulation. The same condition vector yields the same γ/β over the whole image — still global FiLM, merely injected somewhere else.

- **Effect.** Combined with components 1 and 2, **v2ep4 reaches L1 = 0.0949 with no label supervision.**

### 2.4 组件 4：冻结分类器在线监督 / Component 4: Frozen Classifier Supervision

**中文**

- **改了什么。** 把 `her2_cls_regularized.pt`（ResNet18，**AUC 0.8189**）加载进训练循环，**冻结**，对生成图算 CE loss：

  ```python
  loss_gen = 10 * L1 + 5 * SSIM - 1 * cond_contrast + 1 * loss_dir
  ```

- **为什么改。** 组件 1–3 只解决了"生成器对不同 tokens 生成不同图"，**没告诉它哪个方向对**。条件敏感度是一个幅度指标，不是方向指标。

- **效果。这是最大单步提升。** `fake_AUC` 从 **0.4817 到 0.6052**，`SFS` 从 **0.4816 到 0.5555**。

- **代价。** `L1` 从 **0.0949 降到 0.0429** —— 标签监督和条件敏感度在拔河。接受这个代价是因为 SFS 大幅提升。

**English**

- **What changed.** `her2_cls_regularized.pt` (ResNet18, **AUC 0.8189**) is loaded into the training loop, **frozen**, and used to compute a CE loss on generated images:

  ```python
  loss_gen = 10 * L1 + 5 * SSIM - 1 * cond_contrast + 1 * loss_dir
  ```

- **Why.** Components 1–3 only ensure the generator produces *different* images for different tokens. **They say nothing about which direction is correct.** Condition sensitivity is a magnitude metric, not a direction metric.

- **Effect. This is the largest single-step gain.** `fake_AUC` moves from **0.4817 to 0.6052**, `SFS` from **0.4816 to 0.5555**.

- **Cost.** `L1` drops from **0.0949 to 0.0429** — label supervision and condition sensitivity pull against each other. The trade was accepted because SFS rose substantially.

---

## 3. 直方图匹配后处理 / Histogram Matching Post-processing

**中文**

- **做法。** 逐 tile、逐通道，把生成图的分布映射到**同 tile 真实 IHC** 的分布。实现 `hist_match_3c(src_u8, tgt_u8)`，用 `np.interp` 在累积分布上做映射。

- **效果。** v2ep12 的 SFS 从 **0.5555 到 0.5756（+0.020）**。

- **关键限制（重要）。** 匹配增益从 ep25 的 **+0.081** 衰减到 **+0.020**。原因是 v2 训练后颜色分布本身已接近 IHC，**匹配的可操作空间在收窄**。这意味着这一项不会随着继续训练而变大，只会继续变小。

- **评估上界说明。** 当前用的是**同 tile 真实 IHC** 作参考分布。实际推理时若无配对 IHC，需用**训练集平均染色模板**替代，效果会略低。**所以 0.5756 是一个带参考分布的评估上界，不是推理时可达到的数字。**

**English**

- **Method.** Per tile, per channel, map the generated image's distribution onto the distribution of the **real IHC for the same tile**. Implemented as `hist_match_3c(src_u8, tgt_u8)`, using `np.interp` over cumulative distributions.

- **Effect.** v2ep12's SFS goes from **0.5555 to 0.5756 (+0.020)**.

- **Key limitation (important).** The matching gain decays from **+0.081** at ep25 to **+0.020**. The reason is that v2's colour distribution is already close to IHC, so **the headroom for matching is shrinking.** This term will not grow with further training; it will keep shrinking.

- **Evaluation-ceiling caveat.** The reference distribution used here is the **real IHC of the same tile**. At inference time, without paired IHC, it would have to be replaced by a **training-set mean stain template**, with somewhat lower effect. **So 0.5756 is an evaluation ceiling measured with a reference distribution — not a number reachable at inference.**

---

## 4. pairwise ranking loss 的负结果 / The Pairwise Ranking Loss Negative Result

**中文**

这一节记录**为什么失败**，因为它是重要的边界信息，不是无关的失败尝试。

- **改动。** `loss_dir` 从 CE 改为 `softplus(-(p_pos - p_neg) + 0.5).mean()`，权重 **2.0**，CE 降到 **0.3**。从 v2ep12 恢复，lr 减半到 **5e-5**，跑 **15 ep**。

- **结果。** `fake_AUC` 从 **0.6052 降到 0.5981**；`SFS` 从 **0.5756 到 0.5836（+0.008）**。

- **那 +0.008 是什么。** 全部来自阈值校准：slide 12 的 `fake_pos` 从 **0.639 到 0.679**。排序能力本身没有改善。

- **结论。标签监督（CE）把 fake_AUC 推到 0.60 之后，损失函数调整不再改变排序能力。**

- **解释。** pairwise 对"同 batch 内阳性 vs 阴性"的约束比 CE 更松。约束更松，模型倾向于退回更平均的输出 —— 也就是条件敏感度保持、排序能力退化。

**English**

This section records **why it failed**, because that is boundary information, not an incidental dead end.

- **Change.** `loss_dir` switched from CE to `softplus(-(p_pos - p_neg) + 0.5).mean()`, weight **2.0**, CE down to **0.3**. Resumed from v2ep12, learning rate halved to **5e-5**, run for **15 ep**.

- **Result.** `fake_AUC` falls from **0.6052 to 0.5981**; `SFS` goes from **0.5756 to 0.5836 (+0.008)**.

- **What that +0.008 is.** Entirely threshold calibration: slide 12's `fake_pos` moves from **0.639 to 0.679**. Ranking ability itself did not improve.

- **Conclusion. Once label supervision (CE) has pushed fake_AUC to 0.60, changing the loss function no longer changes ranking ability.**

- **Explanation.** A pairwise constraint on "positive vs negative within the batch" is looser than CE. With a looser constraint the model drifts back toward a more averaged output — condition sensitivity retained, ranking degraded.

---

## 5. 完整实验对照表 / Full Experiment Table

**本节数字必须逐字准确。/ Numbers in this section are quoted verbatim.**

| ckpt / 配置 / config | 条件敏感度 L1 / L1 | SSIM | fake_AUC | fake_pos | SFS |
|---|---|---|---|---|---|
| ep25（基线，全局 FiLM / baseline, global FiLM） | 0.0112 | 0.5119 | 0.4885 | 0.847 | 0.4452 |
| ep25 + 直方图匹配 / hist matching | — | — | 0.6027 | 0.728 | 0.5264 |
| v2ep4（多层+空间+数值约束，**无标签** / no labels） | 0.0949 | 0.5559 | 0.4817 | 0.756 | 0.4816 |
| v2ep4（+标签监督，L1 降到 0.0429 / +labels） | 0.0429 | — | — | — | — |
| v2ep8（标签监督 / labels） | — | 0.6561 | 0.5685 | 0.641 | 0.5349 |
| v2ep12（标签监督 / labels） | 0.0467 | 0.6517 | 0.6052 | 0.647 | 0.5555 |
| v2ep12 + 直方图匹配 / hist matching | — | 0.6873 | 0.6045 | 0.576 | **0.5756** |
| pairwise ep15（从 ep12 恢复 / resumed from ep12） | 0.0527 | 0.6498 | 0.5706 | 0.661 | 0.5574 |
| pairwise ep15 + 直方图匹配 / hist matching | — | 0.6852 | 0.5981 | 0.595 | 0.5836 |

**读法 / How to read it**

**中文** —— 三件事同时看：`L1` 在组件 4 之后从 0.0949 掉到 0.0429，但 `fake_AUC` 从 0.4817 涨到 0.6052。**这两个指标是对立的，别指望同时最大化。** 另外 `ep25 + 匹配` 的 fake_AUC（0.6027）比 `v2ep12 + 匹配`（0.6045）只低 0.002 —— 说明匹配本身能显著抬高 AUC，不是 v2 的功劳，这一点容易被误读。

**English** — Read three things together. `L1` falls from 0.0949 to 0.0429 after component 4, while `fake_AUC` rises from 0.4817 to 0.6052. **These two metrics are in tension; do not expect to maximise both.** Also note that `ep25 + matching` reaches fake_AUC 0.6027 versus `v2ep12 + matching` at 0.6045 — only 0.002 apart. Matching alone lifts AUC substantially, so that lift is not attributable to v2. This is easy to misread.

---

## 6. 可复用性说明 / What Transfers to Other Tasks

**中文**

- **组件 1、2 是任何条件生成架构都需要的**，不是本任务特有。一次注入 + 多层上采样几乎必然稀释条件；不做数值约束几乎必然爆炸。
- **组件 3 依赖 token 网格能 reshape 成方阵。** H-Optimus-0 是 16×16，成立。换成 patch 数不是完全平方数的编码器需要插值。
- **组件 4 依赖有一个可信的冻结分类器。** 没有分类器时，组件 1–3 只能把 L1 拉到 **0.09 左右**，SFS 停在 **0.48** —— 这是没有方向信号时的天花板。
- **直方图匹配是推理后处理，与模型无关**，任何 ckpt 都能用。但它的增益随模型变好而衰减（见第 3 节）。

**English**

- **Components 1 and 2 are needed by any conditional generation architecture**, not specific to this task. One injection followed by multi-level upsampling nearly always dilutes the condition; skipping numerical constraints nearly always explodes.
- **Component 3 depends on the token grid reshaping into a square.** H-Optimus-0 is 16×16, so it holds. An encoder whose patch count is not a perfect square would need interpolation.
- **Component 4 depends on having a trustworthy frozen classifier.** Without one, components 1–3 only reach **L1 ≈ 0.09** and SFS stalls at **0.48** — that is the ceiling without a direction signal.
- **Histogram matching is inference-time post-processing, model-independent**, usable with any ckpt. Its gain decays as the model improves (see §3).

---

*内部备忘，2026-10-18。/ Internal memo, 2026-10-18.*
