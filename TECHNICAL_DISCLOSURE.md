# Technical Disclosure — LVMRP

# 技术公开 — LVMRP

**Version:** v0.1.0 · **Release date:** 2026-10-18 · **Author:** IRIXEN Patho Team
**License:** Apache License 2.0 (see [LICENSE](LICENSE)) · **Nature:** defensive publication

> **Status of every claim below.** This document proposes a design. Nothing here is a validated result. Where a mechanism is described, it is described as *the proposal* or *the hypothesis*; where a number appears, it comes from the negative internal experiments listed in [README §4](README.md#4-current-evidence--honest-disclosure) and is quoted verbatim. The core hypothesis — that delta signals are less sensitive to slide-level batch effects than static signals — **remains unvalidated**.
>
> **本文性质。** 以下全部为设计主张，无一是已验证结果。凡描述机制处，均表述为「本方案主张/本方案假设」；凡出现数字处，均取自 [README §4](README.md#4-current-evidence--honest-disclosure) 的负面实验，逐字引用。核心假设——变化量信号对 slide 级批次效应的敏感度低于静态信号——**尚未验证**。

---

## 1. Problem definition / 问题定义

**English**

Let subject `i` be imaged at two or more time points `t_0 < t_1 < …`. Write `X_i^{t_0}` and `X_i^{t_1}` for the H&E whole-slide images at those points, and `c_i` for the clinical covariate vector associated with the interval (treatment regimen, interval length, line of therapy, and any other available metadata).

The proposal seeks a function

```
f( X_i^{t_0}, X_i^{t_1}, c_i )  →  ( ΔM_i , U_i )
```

where

- `ΔM_i` is a **spatially-resolved map of molecular change** over the tissue area present in both slides — for example ΔHER2 expression, Δpathway activity, or Δimmune infiltration;
- `U_i` is a **spatially-resolved uncertainty map**: per-region variance (or an equivalent spread measure) attached to each location of `ΔM_i`.

The output is quantitative and spatial. It contains **three** deliverables and **one** explicit non-deliverable:

1. a delta map `ΔM_i` — numeric, per region;
2. an uncertainty map `U_i` — numeric, per region;
3. a region correspondence between `t_0` and `t_1` (implicit in the map indexing);
4. **not** an RGB stained image. This proposal does not synthesise H&E, IHC, or any other rendered image at any stage.

**中文**

设受试者 `i` 在 `t_0 < t_1 < …` 的至少两个时间点被成像。记 `X_i^{t_0}`、`X_i^{t_1}` 为对应时间点的 H&E 全切片图像，`c_i` 为该时间区间相关的临床协变量向量（治疗方案、间隔长度、治疗线数及其他可得元数据）。

本方案要学习的函数为

```
f( X_i^{t_0}, X_i^{t_1}, c_i )  →  ( ΔM_i , U_i )
```

其中

- `ΔM_i` 是两切片共有组织区域上的**空间分辨分子变化图谱**，例如 ΔHER2 表达、Δ通路活性、Δ免疫浸润；
- `U_i` 是**空间分辨不确定性图**：在 `ΔM_i` 的每个位置上给出该区域的方差（或等价的离散度度量）。

输出是定量的、空间的。交付物有**三项**，明确的**非交付物**有**一项**：

1. 变化图谱 `ΔM_i` —— 数值型，逐区域；
2. 不确定性图 `U_i` —— 数值型，逐区域；
3. `t_0` 与 `t_1` 之间的一致性区域对应（隐含在图谱索引中）；
4. **不是** RGB 染色图像。本方案在任何阶段都不合成 H&E、IHC 或任何其他渲染图像。

---

## 2. System architecture / 系统架构

**English**

### 2.1 Encoder — frozen

The proposal uses a pathology foundation model as a **fixed feature extractor**. The worked embodiment is **H-Optimus-0**:

| Property | Value |
|---|---|
| Backbone | ViT-Giant |
| Patch size | 14 |
| Input tile | 224 × 224 |
| Token tensor | `[N, 261, 1536]` per slide, where 261 = 256 patch tokens + 4 register tokens + 1 CLS token |

The encoder is **frozen end to end**: no gradient flows into it, at any stage, in any configuration described here. This is a design constraint, not a default — it keeps the trainable surface small enough to be defensible on cohorts of the size this problem realistically offers, and it keeps the embodiment swappable.

The encoder is **replaceable**. The proposal is not tied to H-Optimus-0; UNI, Virchow, or Prov-GigaPath are expected to be usable with the same downstream design, subject to matching the token dimension.

### 2.2 Response decoder

```
tokens(X^t0) ─► attention pool ─► z0 ─┐
tokens(X^t1) ─► attention pool ─► z1 ─┤
                                       ├─► condition fusion MLP ─► γ
                       c_i ────────────┘
                                             │
                                             ▼
                          SPADE spatial modulation (multi-resolution)
                                 γ_{h,w}, β_{h,w} at each level
                                             │
                          ┌──────────────────┴──────────────────┐
                          ▼                                     ▼
                    Δ head ─► ΔM        uncertainty head ─► U
```

Step by step:

1. **Attention pooling** reduces each slide's token set to a slide-level vector: `z0`, `z1 ∈ R^d`.
2. **Condition fusion.** `z0`, `z1` and the clinical covariates `c_i` are fused by an MLP into a condition vector `γ`.
3. **SPADE spatial modulation.** `γ` is broadcast into **per-position** scale and shift parameters `γ_{h,w}` and `β_{h,w}`, generated at **multiple resolutions**. The modulation is applied spatially, not globally.
4. **Δ head** emits the molecular change map `ΔM_i`.
5. **Uncertainty head** emits the per-region uncertainty map `U_i`.

### 2.3 Why SPADE and not global FiLM — a design choice tied to an observed failure

The static route in the internal experiments showed **condition sensitivity L1 = 0.0112** ([README §4](README.md#4-current-evidence--honest-disclosure)): changing the condition input barely changed the output. That is the signature of a condition signal that enters once, globally, and is then averaged out by pooling and by the spatial decoder.

**The proposal therefore injects `c_i` as a spatial modulation (SPADE-style, per-position `γ_{h,w}` / `β_{h,w}` at several resolutions) rather than through a global FiLM vector.** The intent is that the condition cannot be averaged away, because it varies per position instead of being a single additive or multiplicative term applied uniformly.

This is a **hypothesis about a remedy**, not a demonstrated fix. No experiment reported here shows that SPADE modulation raises condition sensitivity.

### 2.4 Static route versus this proposal

| | Static virtual staining / molecular prediction | **This proposal (LVMRP)** |
|---|---|---|
| Input | one H&E slide, one time point | H&E at **≥ 2 time points**, same subject |
| Prediction target | absolute molecular state or an image | **Δ molecular state** (change), spatially resolved |
| Condition injection | none, or a single global vector | clinical covariates `c_i` via **SPADE spatial modulation**, multi-resolution |
| Output | RGB image, or a scalar/per-tile score | Δ map + uncertainty map, **never an image** |
| Training data | any H&E cohort | same-subject longitudinal pairs, molecular measurement at each point |

**中文**

### 2.1 编码器 —— 冻结

本方案以病理基础模型作为**固定特征提取器**。实施例取 **H-Optimus-0**：

| 属性 | 值 |
|---|---|
| 主干 | ViT-Giant |
| Patch 尺寸 | 14 |
| 输入 tile | 224 × 224 |
| token 张量 | 每张切片 `[N, 261, 1536]`，其中 261 = 256 个 patch token + 4 个 register token + 1 个 CLS token |

编码器**全程冻结**：此处描述的任何阶段、任何配置下，梯度都不进入编码器。这是设计约束而非默认设置——它把可训练面压到与这类问题现实中可获得的数据规模相称的量级，同时保持实施例可替换。

编码器**可替换**：本方案不绑定 H-Optimus-0，UNI、Virchow、Prov-GigaPath 预期可在同一套下游设计下使用（需匹配 token 维度）。

### 2.2 响应解码器

1. **注意力池化**把每张切片的 token 集合压成切片级向量 `z0`、`z1 ∈ R^d`；
2. **条件融合**：`z0`、`z1` 与临床协变量 `c_i` 经 MLP 融合为条件向量 `γ`；
3. **SPADE 空间调制**：`γ` 被展开为**逐位置**的缩放与偏移参数 `γ_{h,w}`、`β_{h,w}`，在**多个分辨率**上生成并施加，而非全局施加；
4. **Δ 头**输出分子变化图谱 `ΔM_i`；
5. **不确定性头**输出逐区域不确定性图 `U_i`。

### 2.3 为何用 SPADE 而非全局 FiLM —— 针对已观察到的失败模式

内部实验中的静态路线表现为 **条件敏感度 L1 = 0.0112**（见 [README §4](README.md#4-current-evidence--honest-disclosure)）：改变条件输入，输出几乎不变。这正是「条件信号只进入一次、全局施加、随后被池化与空间解码器平均掉」的典型特征。

**因此本方案把 `c_i` 作为空间调制注入**（SPADE 式，在多分辨率上给出逐位置 `γ_{h,w}` / `β_{h,w}`），而不是通过全局 FiLM 向量注入。其意图是使条件无法被平均——因为条件逐位置变化，而不是作为一个统一的加性/乘性项施加。

这是**关于补救手段的假设**，不是已被证明的修复。本文不含任何显示 SPADE 调制能提升条件敏感度的实验。

### 2.4 静态路线与本方案对比

见上表（中英文同一张表，位于本节末）。

| | 静态虚拟染色 / 分子预测 | **本方案（LVMRP）** |
|---|---|---|
| 输入 | 单张 H&E、单时点 | 同一受试者**≥ 2 个时间点**的 H&E |
| 预测目标 | 绝对分子状态或图像 | **Δ 分子状态**（变化量），空间分辨 |
| 条件注入 | 无，或单一全局向量 | 临床协变量 `c_i` 经 **SPADE 空间调制**，多分辨率 |
| 输出 | RGB 图像，或标量/逐 tile 分数 | Δ 图谱 + 不确定性图，**从不输出图像** |
| 训练数据 | 任意 H&E 队列 | 同一受试者纵向配对，每点有分子测量 |

---

## 3. Training pipeline / 训练流程

**English**

### 3.1 Data actually required

A real longitudinal cohort must supply, for each subject `i`:

- **at least two time points** of H&E whole-slide imaging;
- a **molecular measurement at each time point** — this is what makes `ΔM_i` a supervised target rather than a guess;
- **treatment type and interval** between points, recorded as covariates `c_i`.

Where any of these is missing, the pair is unusable for training the delta target. There is no substitution for the per-time-point molecular measurement.

### 3.2 Loss

Four terms, summed:

| Term | Form | Purpose |
|---|---|---|
| Delta regression | `L1(ΔM_pred, ΔM_true)` | accuracy of the change map |
| Heteroscedastic | `NLL` with predicted per-region variance | makes `U_i` calibrated rather than decorative |
| Spatial smoothness | `‖∇Δ‖₁` | suppresses implausible per-tile speckle |
| Representation non-collapse | `‖z1 − z0‖₂` (as a penalty on degenerate collapse) | prevents the trivial solution where the model ignores the time axis |

The fourth term exists because a delta model that outputs a constant satisfies the first term cheaply on a cohort with weak signal. It is a guard, not a guarantee.

### 3.3 Simulated longitudinal warm-start

Before real longitudinal data exists, the architecture can be exercised on **simulated longitudinal** pairs built from a static dataset: assign one slide of a subject to `t_0` and another to `t_1`, producing pseudo-pairs `A → t_0, B → t_1`.

> **This validates architecture feasibility only. It does not validate the biological hypothesis.** A pseudo-pair's "delta" is, to a first approximation, the *difference between two slides*, which is exactly the slide-level batch effect the method is supposed to be robust to. Pseudo-pair deltas are therefore contaminated by slide fingerprints, and any metric computed on them says nothing about whether real Δ is predictable.

This limitation is stated here deliberately, because a warm-start result is easy to over-read.

### 3.4 Optimisation details

| Setting | Value |
|---|---|
| Trainable decoder size | 10–50 M parameters |
| Optimiser | AdamW |
| Learning rate | 1e-4 |
| Weight decay | 1e-2 |
| Early stopping | slide-level Spearman on the validation split |

**中文**

### 3.1 真实所需数据

真实纵向队列须为每个受试者 `i` 提供：

- **至少两个时间点**的 H&E 全切片；
- **每个时间点各有一次分子测量** —— 这决定了 `ΔM_i` 是有监督目标而不是猜测；
- 两点之间的**治疗类型与间隔**，作为协变量 `c_i` 记录。

任一项缺失，该配对即不能用于训练 Δ 目标。**逐时间点的分子测量没有任何替代品。**

### 3.2 损失函数

四项相加：

| 项 | 形式 | 作用 |
|---|---|---|
| Δ 回归 | `L1(ΔM_pred, ΔM_true)` | 变化图谱的准确度 |
| 异方差 | 带逐区域预测方差的 `NLL` | 使 `U_i` 具备校准意义而非装饰 |
| 空间平滑 | `‖∇Δ‖₁` | 抑制不合理的逐 tile 斑点 |
| 表示不塌缩 | `‖z1 − z0‖₂`（作为对退化塌缩的惩罚） | 防止模型忽略时间轴的平凡解 |

第四项的动因：在信号微弱的队列上，一个恒输出常数的 Δ 模型可以廉价地满足第一项。它是护栏，不是保证。

### 3.3 模拟纵向 warm-start

在真实纵向数据出现之前，可用静态数据集构造**模拟纵向**配对：把同一受试者的一张切片指派为 `t_0`、另一张为 `t_1`，形成伪对 `A → t_0, B → t_1`。

> **这只验证架构可行性，不验证生物学假设。** 伪对的「变化量」在一阶近似下就是*两张切片之差*，而这恰是本方法本应对其鲁棒的 slide 级批次效应。因此伪对的 Δ 被切片指纹污染，在其上得到的任何指标都不能说明真实 Δ 是否可预测。

这条限制写在这里是刻意的，因为 warm-start 的结果很容易被过度解读。

### 3.4 优化细节

见上表（中英文同一张表）。

| 设置 | 值 |
|---|---|
| 可训练解码器规模 | 10–50 M 参数 |
| 优化器 | AdamW |
| 学习率 | 1e-4 |
| 权重衰减 | 1e-2 |
| 早停依据 | 验证集 slide 级 Spearman |

---

## 4. Evaluation protocol / 评估协议

**English**

### 4.1 Primary metrics

| Metric | What it measures |
|---|---|
| Slide-level Spearman of Δ | rank agreement between predicted and measured change, per subject |
| Direction accuracy | share of subjects whose predicted sign of change matches the measured sign |
| Uncertainty calibration | whether `U_i` predicts the actual error — calibration curve / coverage |

### 4.2 Diagnostics that must be reported

A delta model can score well for the wrong reason. These three diagnostics must accompany any primary metric:

| Diagnostic | Failure threshold | Reading |
|---|---|---|
| Slide-identity AUC on the decoder's Δ representation | **> 0.9 = failure** | the delta representation still encodes *which slide it came from*, i.e. the batch fingerprint survived |
| Domain-discrimination AUC (train vs val) | high = failure | the representation separates splits rather than biology |
| Spatial variance of Δ | **≈ 0 = failure** | the model emits a near-constant map |

The first diagnostic is the load-bearing one. In the static experiments the analogous slide-identity AUC was **1.0000** and the domain AUC **0.9995** ([README §4](README.md#4-current-evidence--honest-disclosure)); a method that cannot push the delta representation's slide-identity AUC down is not measuring biology.

### 4.3 Holdout design

Splits are **by subject**, and **both time points of a subject must land on the same side** of the split. Splitting by slide would place a subject's `t_0` in train and `t_1` in test, which leaks the subject and destroys the pairing that the metric depends on.

Results are reported as **leave-one-subject-out (LOSO)** mean ± standard deviation.

**中文**

### 4.1 主指标

| 指标 | 衡量内容 |
|---|---|
| Δ 的 slide 级 Spearman | 逐受试者，预测变化量与测量变化量的秩一致性 |
| 方向准确率 | 预测变化符号与实测符号一致的受试者比例 |
| 不确定性校准 | `U_i` 是否预测了真实误差 —— 校准曲线 / 覆盖率 |

### 4.2 必须报告的诊断指标

Δ 模型可能因为错误的原因而得分好看。以下三项必须与任何主指标同时报告：

| 诊断 | 失败阈值 | 含义 |
|---|---|---|
| 解码器 Δ 表示上的 slide 身份 AUC | **> 0.9 即失败** | Δ 表示仍然编码了「来自哪张切片」，即切片指纹存活 |
| 域判别 AUC（train vs val） | 高即失败 | 表示区分的是划分而不是生物学 |
| Δ 的空间方差 | **≈ 0 即失败** | 模型输出近似常数图 |

第一项是承重的。静态实验中对应的 slide 身份 AUC 为 **1.0000**、域判别 AUC 为 **0.9995**（见 [README §4](README.md#4-current-evidence--honest-disclosure)）；一个无法把 Δ 表示的 slide 身份 AUC 压下去的方法，测的就不是生物学。

### 4.3 留出策略

划分**按受试者**进行，且**同一受试者的两个时间点必须同在 train 或同在 test**。按切片划分会把某人的 `t_0` 放进训练、`t_1` 放进测试，既泄露受试者，也破坏指标所依赖的配对结构。

结果以**留一受试者（LOSO）** 的均值 ± 标准差报告。

---

## 5. Distinguishing technical features / 区别技术特征

**English**

The following six features together delimit what this disclosure puts in the public domain. Each is stated as a feature of the proposal, not as a validated benefit.

1. **Two or more time points as input.** The method consumes H&E from the same subject at ≥ 2 time points; a single-slide input is outside its scope.
2. **Δ prediction as the target.** The learned target is the change in molecular state, not the state itself.
3. **Non-image output.** The output is a quantitative delta map plus an uncertainty map; no RGB image is produced at any stage.
4. **SPADE + time conditioning.** Clinical covariates are injected as per-position, multi-resolution spatial modulation rather than a global FiLM vector.
5. **Simulated longitudinal warm-start.** Pseudo-pairs assembled from a static cohort are used explicitly as an architecture-feasibility step, with the contamination caveat attached.
6. **Slide-identity AUC as a failure diagnostic.** The delta representation is audited for residual slide identity, with > 0.9 treated as failure.

**中文**

以下六项共同界定本公开置入公共领域的范围。每项都表述为「本方案的特征」，而非「已验证的收益」。

1. **≥ 2 个时间点作为输入**：方法消费同一受试者至少两个时间点的 H&E；单切片输入不在其范围内。
2. **以 Δ 为预测目标**：学习目标是分子状态的变化量，而非状态本身。
3. **非图像输出**：输出为定量变化图谱 + 不确定性图；任何阶段都不产生 RGB 图像。
4. **SPADE + 时间条件**：临床协变量以逐位置、多分辨率的空间调制注入，而非全局 FiLM 向量。
5. **模拟纵向 warm-start**：用静态队列拼出的伪对，明确作为架构可行性步骤使用，并附带污染说明。
6. **以 slide 身份 AUC 作为失败诊断**：对 Δ 表示审计残存切片身份，> 0.9 即视为失败。

---

## 6. Known failure modes and open questions / 已知失败模式与开放问题

**English**

**Failure modes** — observed in the static route, expected to threaten this one:

- **Slide fingerprint.** Slide identity is highly predictable from features (AUC **1.0000**), and colour normalisation does not remove it (**0.9994 → 0.9995**). A delta model that inherits this representation inherits the problem.
- **Condition collapse.** Condition sensitivity L1 **0.0112**: the conditioning input can be effectively ignored. SPADE modulation is the proposed countermeasure and is itself unvalidated.
- **Label noise.** Per-time-point molecular measurements carry their own assay variability, which propagates straight into `ΔM_true`.

**Open questions** — deliberately left open:

1. **How long an interval is usable?** Too short and Δ is dominated by assay noise; too long and intervening biology dominates.
2. **One model across treatments, or one per treatment class?** The proposal does not decide this; it may be that the condition vector is insufficient and per-treatment heads are required.
3. **How is spatial ground truth for Δ obtained at all?** A per-time-point molecular measurement is typically bulk or limited-plex; obtaining *spatially resolved* Δ ground truth is an unsolved data problem, and it gates the entire evaluation.

**中文**

**失败模式** —— 在静态路线中已观察到，预期同样威胁本方案：

- **切片指纹**：切片身份可从特征高度预测（AUC **1.0000**），颜色归一化无法去除（**0.9994 → 0.9995**）。继承了该表示的 Δ 模型会继承该问题。
- **条件塌缩**：条件敏感度 L1 **0.0112**，条件输入可能被实际忽略。SPADE 调制是提出的对策，其本身尚未验证。
- **标签噪声**：逐时间点分子测量自带检测变异，会直接传播进 `ΔM_true`。

**开放问题** —— 刻意留白：

1. **多长的间隔可用？** 太短则 Δ 被检测噪声主导；太长则中间发生的生物学主导。
2. **跨治疗单一模型，还是每类治疗一个模型？** 本方案未做决定；有可能条件向量不足，必须按治疗分头。
3. **空间 Δ 的真值究竟如何获得？** 逐时间点的分子测量通常是 bulk 或低 plex；获得*空间分辨*的 Δ 真值本身是未解决的数据问题，并且它卡住整个评估。

---

## 7. Implementability / 可实施性声明

**English**

The proposal is implementable with public components only:

- **H-Optimus-0** is publicly available and Apache-2.0 licensed, and serves as the frozen encoder in the worked embodiment;
- **PyTorch** provides everything needed for attention pooling, SPADE modulation and the two heads;
- the **simulated longitudinal warm-start** requires nothing beyond a static H&E cohort, so the architecture can be reproduced and inspected before any longitudinal data is available;
- no proprietary dataset is required to exercise the architecture.

Reproducing the architecture is therefore straightforward. Reproducing a **positive** result is not: that requires real longitudinal data with per-time-point molecular measurements, which this disclosure does not have.

**中文**

本方案仅用公开组件即可实施：

- **H-Optimus-0** 公开可得且为 Apache-2.0 许可，在实施例中作为冻结编码器；
- **PyTorch** 提供了注意力池化、SPADE 调制与两个头所需的全部能力；
- **模拟纵向 warm-start** 只需一个静态 H&E 队列，因此在任何纵向数据出现之前，架构即可被复现与检视；
- 练习该架构不需要任何专有数据集。

因此**复现架构**是直接的。复现一个**正向结果**不是：那需要带逐时间点分子测量的真实纵向数据，而本公开并不具备。

---

*This document is a defensive publication. It is published to establish prior art and to prevent any party from obtaining exclusive rights over the subject matter described. The publishers assert no patent rights over it.*
