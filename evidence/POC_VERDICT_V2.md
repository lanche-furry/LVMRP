# IRIXEN / Patho-1 — LVMRP v2 POC 结论：17-slide 上限的量化判断

# IRIXEN / Patho-1 — LVMRP v2 POC Verdict: Quantifying the 17-slide Ceiling

> **内部技术备忘 / Internal technical memo.** 不是交付文档。所有数字引自实验日志，不做改写。
>
> **Internal memo, not a deliverable.** Numbers quoted from the experiment log, unmodified.

---

## 1. 一句话结论 / One-line Verdict

**中文**

> 在 17-slide HER2match 数据 + H-Optimus-0 tokens 上，静态 H&E→IHC 虚拟染色的 **SFS 上限约 0.58，fake_AUC 上限约 0.60**。距离可用（SFS ≥ 0.65、fake_AUC ≥ 0.70）还差 0.07–0.10。**这个差距不是训练技巧能填的。**

**English**

> On the 17-slide HER2match data with H-Optimus-0 tokens, static H&E→IHC virtual staining caps out at **SFS ≈ 0.58 and fake_AUC ≈ 0.60**. Usable performance (SFS ≥ 0.65, fake_AUC ≥ 0.70) is still **0.07–0.10 away**. **That gap is not closable with training tricks.**

---

## 2. 三层证据 / Three Layers of Evidence

### 2.1 证据 1：架构改造能拉条件敏感度，但拉不动 AUC / Evidence 1: architecture raises condition sensitivity, not AUC

**中文**

- 多层 FiLM + 空间注入把 `L1` 从 **0.0112** 拉到 **0.0949（8.5×）**。
- 但 `fake_AUC` 从 **0.4885 到 0.4817**，**没动**（甚至略降）。
- 说明"生成器对不同 tokens 生成不同图"和"这些差异指向正确的 HER2 方向"是**两件事**。前者是幅度，后者是方向。架构改造只买到幅度。

**English**

- Multi-layer FiLM plus spatial injection lifts `L1` from **0.0112** to **0.0949 (8.5×)**.
- Yet `fake_AUC` moves from **0.4885 to 0.4817** — **no change**, and slightly down.
- This shows that "the generator produces different images for different tokens" and "those differences point in the correct HER2 direction" are **two different problems**. The first is magnitude; the second is direction. The architectural rework bought only magnitude.

### 2.2 证据 2：标签监督是唯一能拉 AUC 的手段，但有上限 / Evidence 2: label supervision is the only lever for AUC, and it saturates

**中文**

- 加冻结分类器 CE 监督后，`fake_AUC` 从 **0.4817 到 0.6052（+0.12）**。
- 继续训练到 ep15、换 pairwise loss，AUC **不再上升**（0.6052 → 0.5981，反而下降）。
- 说明 **0.60 是这个 tokens 表示的上限**，不是优化没到位的中间值。

**English**

- Adding frozen-classifier CE supervision moves `fake_AUC` from **0.4817 to 0.6052 (+0.12)**.
- Training on to ep15 and switching to a pairwise loss does **not** raise AUC further (0.6052 → 0.5981, actually down).
- This says **0.60 is the ceiling of this token representation**, not an intermediate value that more optimisation would pass.

### 2.3 证据 3：slide 16 和 slide 1 是系统性失败点 / Evidence 3: slides 16 and 1 are systematic failure points

**中文**

| slide | 真实标签 | `fake_pos` | 判读 |
|---|---|---|---|
| slide 16 | 阴性 / negative | **0.975** | 几乎完全误判为阳性 |
| slide 1 | 阴性 / negative | **0.720** | 明显偏阳性 |
| slide 8 | 阴性 / negative | **0.147** | 表现良好 |

说明模型**不是完全学不到阴性** —— slide 8 上它做得很好。而是**对某些染色批次的阴性失效**。这直接回到 slide 指纹主导的老问题：模型学到的"阴性长相"是特定批次的阴性长相。

**English**

| slide | true label | `fake_pos` | reading |
|---|---|---|---|
| slide 16 | negative | **0.975** | almost entirely misclassified as positive |
| slide 1 | negative | **0.720** | clearly biased positive |
| slide 8 | negative | **0.147** | behaves well |

So the model is **not** unable to learn negatives — on slide 8 it does well. It fails on **the negatives of particular staining batches**. This points straight back to slide-fingerprint dominance: what the model learned as "a negative looks like" is the negative of a specific batch.

---

## 3. 与 ep25 基线的对照 / Against the ep25 Baseline

| 阶段 / stage | SFS | 相对 ep25 / vs ep25 |
|---|---|---|
| ep25 raw | 0.4452 | — |
| ep25 + 匹配 / matching | 0.5264 | +0.081 |
| v2ep12 raw | 0.5555 | +0.110 |
| **v2ep12 + 匹配 / matching** | **0.5756** | **+0.130** |

**中文**

关键句：**SFS 从 0.4452 到 0.5756，提升 29%，其中架构改造贡献 +0.11，直方图匹配贡献 +0.02。**

需要说清的一点：**+0.13 里有 0.081 是后处理**。如果只看模型本身，从 0.4452 到 0.5555 是 +0.110，即约 25%。而直方图匹配的增益在缩小（ep25 上 +0.081，v2 上 +0.020），说明它是一项会随模型变好而失效的增益，不能计入长期路线。

**English**

The key sentence: **SFS goes from 0.4452 to 0.5756, a 29% improvement, of which the architectural rework contributes +0.11 and histogram matching +0.02.**

One thing to state plainly: **0.081 of that +0.13 is post-processing.** Looking at the model alone, 0.4452 → 0.5555 is +0.110, roughly 25%. And the matching gain is shrinking (+0.081 on ep25, +0.020 on v2), so it is a benefit that decays as the model improves and should not be counted in the long-term plan.

---

## 4. 下一步的明确判断 / Next Steps: What Not To Do, What To Do

### 4.1 不再做 / Not doing

**中文**

- **不在 17 slide 上继续训练或调损失。** 上限已经测出来了，继续调只是在这个上限下反复。
- **不换损失函数。** CE、pairwise 都已试过。CE 更好，pairwise 更差，两者都不改变 0.60 的排序上限。
- **不加 epoch。** ep8→ep12 的增益 **+0.021**，ep12→ep15 的增益 **+0.008**，已饱和。

**English**

- **No more training or loss tuning on 17 slides.** The ceiling has been measured; further tuning only oscillates beneath it.
- **No more loss-function swaps.** CE and pairwise have both been tried. CE is better, pairwise is worse, and neither moves the 0.60 ranking ceiling.
- **No more epochs.** The ep8→ep12 gain is **+0.021**, the ep12→ep15 gain is **+0.008** — saturated.

### 4.2 要做 / Doing

**中文**

1. **ACROBAT 重训。** 750 例训练集，**CC BY 4.0 可商用**，H&E–HER2 IHC 配对。用 `train_lvmrp_v2.py` 的组件 1–4，把 `RealPairedDataset` 的路径和 `SLIDE_SCORE` 换成 ACROBAT 的映射。**这是验证"是不是 17 slide 太少"的直接方法** —— 如果 ACROBAT 上上限显著抬高，结论就是数据量；如果仍是 0.6 左右，结论是方法或 tokens。
2. **换编码器（如果 ACROBAT 也不够）。** H-Optimus-0 是 ViT-Giant。如果 tokens 本身信息量不足，换更大的模型（如 Prov-GigaPath）或换多尺度特征。
3. **LVMRP 纵向验证。** 等客户数据，**不改架构**。组件 1–4 已经在静态任务上验证了机制，纵向任务沿用。

**English**

1. **Retrain on ACROBAT.** 750 training cases, **CC BY 4.0, commercially usable**, paired H&E–HER2 IHC. Use components 1–4 from `train_lvmrp_v2.py`, replacing `RealPairedDataset`'s paths and `SLIDE_SCORE` with the ACROBAT mapping. **This is the direct test of "are 17 slides simply too few?"** — if the ceiling rises materially on ACROBAT, the answer is data volume; if it stays near 0.6, the answer is method or tokens.
2. **Swap the encoder (if ACROBAT is not enough).** H-Optimus-0 is ViT-Giant. If the tokens themselves lack information, move to a larger model (e.g. Prov-GigaPath) or to multi-scale features.
3. **LVMRP longitudinal validation.** Wait for customer data, **do not change the architecture**. Components 1–4 have demonstrated their mechanism on the static task; carry them over.

### 4.3 一句话 / One line

**中文**

技术组件是实的，数据上限是量的，**下一个窗口的起点是 ACROBAT，不是继续调 ep25**。

**English**

The components are real; the data ceiling is quantitative. **The next window starts at ACROBAT, not with more ep25 tuning.**

---

*内部备忘，2026-10-18。/ Internal memo, 2026-10-18.*
