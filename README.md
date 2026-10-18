# LVMRP — Longitudinal Virtual Molecular Response Profiling

# LVMRP — 纵向虚拟分子响应图谱

> **Defensive publication.** Everything in this repository is published so that it enters the public domain of technical knowledge. The publishers do **not** retain, and do **not** intend to assert, any patent claim over the subject matter disclosed here. This document also claims **no priority and no originality**: it does not assert that any element described here is new, nor that the combination has never been described. It exists so that no party can obtain exclusivity over the disclosed approach — nothing more.
>
> **防御性公开。** 本仓库的全部内容公开即视为进入公共技术知识领域。发布者**不保留**、也**不打算主张**对本文所公开内容的任何专利权。本文件同时**不主张任何优先权，也不主张任何独创性**：它既不断言其中任一要素是新的，也不断言该组合从未被描述过。它的作用只是使任何主体都无法对该方案取得排他权——仅此而已。

> **Note on acronym.** LVMRP here stands for Longitudinal Virtual Molecular Response Profiling. It is unrelated to Laparoscopic Ventral Mesh Rectopexy, which uses the same acronym in a different field.
>
> **缩写说明。** 此处的 LVMRP 指 Longitudinal Virtual Molecular Response Profiling（纵向虚拟分子响应图谱），与另一领域中同缩写的 Laparoscopic Ventral Mesh Rectopexy（腹腔镜腹壁网片直肠固定术）无关。

[![License: Apache-2.0](https://img.shields.io/badge/license-Apache--2.0-blue.svg)](LICENSE)
[![Status: proposed, unvalidated](https://img.shields.io/badge/status-proposed%2C%20unvalidated-lightgrey.svg)](#4-current-evidence--honest-disclosure)
[![Version](https://img.shields.io/badge/version-v0.1.0-informational.svg)](https://github.com/lanche-furry/LVMRP/releases)

| Field / 项 | Value / 值 |
|---|---|
| Repository / 仓库 | `LVMRP` |
| Full name (EN) | Longitudinal Virtual Molecular Response Profiling |
| 全称（中） | 纵向虚拟分子响应图谱 |
| Internal codename / 内部代号 | IRIXEN-Delta / IRIXEN-Response |
| Version / 版本 | `v0.1.0` |
| Release date / 发布日 | 2026-10-18 |
| Author / 作者 | IRIXEN Patho Team |
| Code license / 代码许可 | Apache License 2.0 |
| Nature / 性质 | Defensive publication / 防御性公开 |

---

## One-sentence definition / 一句话定义

**中文：**

> 一种从同一受试者至少两个时间点的 H&E 全切片图像中，预测空间分辨的分子状态变化量（如 ΔHER2 表达、Δ通路活性、Δ免疫浸润）的方法；输出为定量变化图谱及不确定性图，不生成 RGB 染色图像。

**English:**

> A method for predicting spatially-resolved **changes** in molecular state (e.g. ΔHER2 expression, Δpathway activity, Δimmune infiltration) from H&E whole-slide images acquired at **two or more time points** from the same subject, outputting quantitative delta maps and uncertainty maps, **without generating RGB stained images**.

---

## 1. Why this direction / 为什么需要这个方向

**中文**

现有计算病理工作绝大多数在回答一个问题：**一张静态的 H&E 切片，对应怎样的分子状态？** 虚拟免疫组化、空间分子预测、H&E→定量蛋白回归都属于这一类。它们的技术形态是「单时点输入 → 静态分子读数」。

药物研发真正关心的却是另一个问题：**治疗有没有改变分子状态？** 这个问题在数据形态上是纵向的（同一受试者、≥2 时间点），在输出形态上是**差分**的（Δ 而非绝对值），在评估上是**配对**的（同一受试者的 t0 与 t1 必须同时进入训练或同时留出）。

两条路线不是同一方案的不同参数，而是三个层面都不同：

- **技术方案**：静态路线不需要时间条件注入，也不需要 Δ 头与不确定性头；
- **数据要求**：静态路线可用任意 H&E 队列；纵向路线要求同一受试者的重复取样与每点分子测量；
- **评估协议**：静态路线按切片或受试者留出；纵向路线必须按受试者成组留出，否则 Δ 的配对结构被破坏，指标失去意义。

**English**

Almost all published computational-pathology work answers one question: *given a single static H&E slide, what is the molecular state?* Virtual immunohistochemistry, spatial molecular prediction and H&E-to-quantitative-protein regression all belong to this family. Their technical shape is **one time point in, one static molecular readout out**.

Drug development asks a different question: **did the treatment change the molecular state?** That question is longitudinal in its data shape (same subject, two or more time points), **differential** in its output shape (Δ rather than an absolute level), and **paired** in its evaluation (the t0 and t1 of one subject must both fall in train or both in test).

These are not two settings of one design; they differ on three axes:

- **Method**: the static family needs no time conditioning, and no delta or uncertainty head;
- **Data**: the static family accepts any H&E cohort; the longitudinal route requires repeat sampling from the same subject with a molecular measurement at each point;
- **Evaluation**: the static family splits by slide or by subject; the longitudinal route must split by subject as a group, otherwise the pairing structure of Δ collapses and the metrics stop meaning anything.

---

## 2. Method overview / 技术方案概要

**中文**

两个时间点的 H&E 全切片分别通过**同一个完全冻结的**病理基础模型编码，得到 patch 级特征序列；经注意力池化得到切片级表示 `z0`、`z1`；再把临床协变量 `c`（治疗类型、时间间隔等）融合成条件向量 `γ`；`γ` 通过 **SPADE 式空间调制**在每个空间位置上生成缩放/偏移参数，驱动 Δ 头输出**空间分辨的分子变化图谱**，同时由不确定性头输出**逐区域方差**。

关键取舍：条件信号走**空间调制**而不是全局 FiLM——理由见 [TECHNICAL_DISCLOSURE.md](TECHNICAL_DISCLOSURE.md) 第 2 节，直接针对静态方案中「条件信号被全局平均化」这一失败模式（条件敏感度 L1 = **0.0112**）。

```
   H&E at t0 ──► frozen pathology foundation model ──► attention pool ──► z0 ─┐
                                                                               ├─► condition
   H&E at t1 ──► frozen pathology foundation model ──► attention pool ──► z1 ─┘    fusion MLP
                                                                                        │
   clinical covariates c (treatment, interval) ────────────────────────────────────────┤
                                                                                        ▼
                                                                              γ  (condition vector)
                                                                                        │
                                                                    SPADE spatial modulation (multi-res)
                                                                                        │
                                                        ┌───────────────────────────────┴───────────────┐
                                                        ▼                                               ▼
                                              Δ head → Δ molecular maps              uncertainty head → uncertainty maps
                                                        │                                               │
                                                        └────────────► output: maps, not RGB ◄──────────┘
```

**English**

Two H&E whole-slide images are encoded by the **same, fully frozen** pathology foundation model into patch-token sequences; attention pooling yields slide-level representations `z0` and `z1`. Clinical covariates `c` (treatment type, interval) are fused into a condition vector `γ`, which drives **SPADE-style spatial modulation** — per-position scale and shift — feeding a Δ head that emits a **spatially-resolved molecular change map**, plus an uncertainty head that emits **per-region variance**.

The deliberate design choice is that the condition enters through **spatial modulation** rather than global FiLM. The rationale is in [TECHNICAL_DISCLOSURE.md](TECHNICAL_DISCLOSURE.md) §2, and it targets precisely the failure mode observed in the static route: a condition signal that is averaged away (condition sensitivity L1 = **0.0112**).

---

## 3. Boundary with prior art / 与现有技术的边界

**中文**

| 现有技术类别 | 本方案不…… |
|---|---|
| H&E → 虚拟 IHC **图像** | 不生成 RGB 染色图像，不依赖配对的 H&E–IHC 训练数据，预测的是 Δ 而非图像 |
| **静态**分子状态预测 | 不接受单时点输入即给出绝对分子读数；预测目标定义为变化量 |
| **非配对** H&E/IHC 虚拟染色 | 输出不是图像；训练依赖同一受试者的纵向配对，而非跨受试者的非配对域转换 |
| **序列切片**（Z 轴）→ 虚拟 IHC | 时间维度是真实生物学时间（治疗前后），不是同一块的 Z 轴层切；输出不是图像 |
| **纵向 H&E/IHC → 临床疗效预测**（`CN113316823A`、`EP3857564A1`、`CN121747954A`、`US2025299802A1`） | 本方案**不输出**患者级别的疗效预测；输出终止于空间 Δ 图谱，临床解读由使用者完成 |

> 最后一行是全文最需要看清的一行：与最接近的现有技术的区别**不在**纵向输入，而在**输出是否走到临床标签**。详见 [PRIOR_ART §4b](PRIOR_ART.md)。

**English**

| Prior-art family | This proposal does **not**… |
|---|---|
| H&E → virtual IHC **images** | …generate RGB stained images, rely on paired H&E–IHC training data, or predict an image rather than Δ |
| **Static** molecular-state prediction | …accept a single time point and emit an absolute molecular level; the target is defined as a change |
| **Unpaired** H&E/IHC virtual staining | …output an image, or train on cross-subject unpaired domain translation instead of same-subject longitudinal pairs |
| **Serial-section** (Z-axis) → virtual IHC | …treat depth as time; the time axis is real biological time (pre/post treatment), and the output is not an image |
| **Longitudinal H&E/IHC → clinical outcome prediction** (`CN113316823A`, `EP3857564A1`, `CN121747954A`, `US2025299802A1`) | …output a patient-level efficacy prediction; the output terminates at the spatial Δ map, and clinical interpretation is left to the user |

> That last row is the one to read carefully: the difference from the closest prior art is **not** the longitudinal input — it is **whether the output reaches a clinical label**. See [PRIOR_ART §4b](PRIOR_ART.md).

Full analysis: [PRIOR_ART.md](PRIOR_ART.md).

---

## 4. Current evidence — honest disclosure / 当前验证状态（诚实披露）

**中文**

本节是这份防御性公开的技术可信度基础，因此**负面证据全部保留**，不做修饰。以下数字来自内部实验，逐字列出。

**（a）静态 H&E → HER2 分类在 17 张切片的数据上失败**

| 指标 | 数值 |
|---|---|
| 验证集 tile 级 AUC | **0.27** |
| 验证集 slide 级 AUC | **0.25** |
| slide 身份分类 AUC | **1.0000** |
| train-vs-val 域判别 AUC | **0.9995** |
| Macenko 归一化后 slide 身份 AUC | **0.9994 → 0.9995**（无效） |
| 静态虚拟染色 SFS | **0.445** |
| 直方图匹配后 SFS | **0.526** |
| 条件敏感度 L1 | **0.0112** |

**（b）根因判读**

slide 身份可以被完美预测（AUC **1.0000**），train-vs-val 也能被域判别器几乎完美区分（AUC **0.9995**）。这说明特征空间里占主导的是**切片级批次/染色指纹**，而不是生物学信号。AUC **0.27** / **0.25** 显著低于 0.5，是被批次方向系统性反向驱动的结果，不是随机噪声。

**（c）Macenko 归一化不解决问题**

做颜色归一化后 slide 身份 AUC 从 **0.9994** 只变到 **0.9995**，等于没动。说明该批次效应不是简单的染色强度/色偏，归一化触及不到。

**（d）静态虚拟染色同样受限**

SFS **0.445**，直方图匹配后到 **0.526**，仍不可用。

**（e）条件信号被平均化**

条件敏感度 L1 **0.0112**：改变条件输入，输出几乎不变。这正是本方案采用 SPADE 空间调制（而非全局 FiLM）的直接动因。

**（f）核心假设尚未验证**

本方案的核心假设是：**变化量信号对 slide 级批次效应的敏感度低于静态信号**。

> **该假设尚待真实纵向数据验证。** 目前没有任何真实纵向队列结果支持它。若该假设不成立，本方案不成立。

**English**

This section is the technical credibility basis of a defensive publication, so **all negative evidence is retained** and nothing is smoothed over. The numbers below come from internal experiments and are quoted verbatim.

**(a) Static H&E → HER2 classification failed on a 17-slide dataset**

| Metric | Value |
|---|---|
| Validation tile-level AUC | **0.27** |
| Validation slide-level AUC | **0.25** |
| Slide-identity classification AUC | **1.0000** |
| Train-vs-val domain discrimination AUC | **0.9995** |
| Slide-identity AUC after Macenko normalisation | **0.9994 → 0.9995** (no effect) |
| Static virtual staining SFS | **0.445** |
| SFS after histogram matching | **0.526** |
| Condition sensitivity L1 | **0.0112** |

**(b) Root-cause reading**

Slide identity is perfectly predictable (AUC **1.0000**) and train-vs-val is almost perfectly separable (AUC **0.9995**). What dominates the feature space is therefore a **slide-level batch / staining fingerprint**, not biology. The AUC values of **0.27** and **0.25** are significantly *below* 0.5, which is the signature of a systematically inverted batch direction — not random noise.

**(c) Macenko normalisation does not fix it**

After colour normalisation the slide-identity AUC moves only from **0.9994** to **0.9995** — i.e. not at all. The artifact is not a simple stain-intensity or colour-cast shift, and normalisation cannot reach it.

**(d) Static virtual staining is limited for the same reason**

SFS **0.445**, rising only to **0.526** after histogram matching — still unusable.

**(e) The condition signal is averaged away**

Condition sensitivity L1 **0.0112**: changing the condition input barely changes the output. This is the direct motivation for SPADE spatial modulation in place of global FiLM.

**(f) The core hypothesis is unvalidated**

The core hypothesis is that **delta signals are less sensitive to slide-level batch effects than static signals**.

> **This hypothesis still awaits validation on real longitudinal data.** No real longitudinal cohort supports it today. If the hypothesis fails, the method fails.

---

## 4b. Implementation-risk note / 实施风险提示

**中文**

本公开是防御性公开，**不构成 FTO（自由实施）意见**。

LVMRP 的**原始定位**——纵向 H&E → 空间 Δ 分子图谱 → 治疗反应——与部分现有专利在「纵向 + 分子变化 + 治疗反应」这一层面存在**实质重叠**，其中最贴近的是 `CN113316823A`（罗氏 / Pathomiq，及其 EP 同族 `EP3857564A1`，**当前状态 Pending**）与 `CN121747954A`。`CN113316823A` 的权利要求 1 实质为：分子数据 + 活检图像数据（含 H&E）→ 机器学习**临床预测**（铂类耐药、存活率），说明书 [0022] 更明确覆盖"随时间的患者治疗反应分析"。**防御性公开阻止的是「他人就你公开的内容获得专利」，并不阻止「你实施时落入他人已有专利」——这两件事必须分开处理。**

当前设计通过把**输出限定为空间 Δ 图谱、不生成患者级别的临床预测**来降低字面重叠风险（见 [TECHNICAL_DISCLOSURE §1](TECHNICAL_DISCLOSURE.md#1-problem-definition--问题定义) 的否定性限定、§5 的区别特征排序与 §5.1 的权利要求式边界），但：

- 这是基于**当前检索到的权利要求字面**所做的边界判断，**不是 FTO 结论**，也不能保证未来不被其他专利覆盖；
- `CN113316823A` 状态为 **Pending**，最终授权权利要求可能与公开版本不同；
- `CN121747954A` 的权利要求全文尚未逐条核查；
- 「不输出患者级预测」这一规避点，在实际实施中是否会被认定为「实质上的治疗反应预测」（例如使用者直接把图谱的聚合读数当作预测使用），**存在解释空间**。

**因此：若要真正实施 LVMRP（而非仅作防御性公开），在投入研发之前，应针对 `CN113316823A` 的权利要求做一次正式 FTO 分析。**

> **本公开是防御性公开，不构成 FTO 意见。文档中的"不输出患者级预测"是产品设计约束，不是法律屏障。产品化前必须针对 CN113316823A、CN121747954A 做 knock-out search 或完整 FTO 分析。防御性公开阻止的是他人就本公开内容获得专利，不能替代对已授权专利的实施风险评估。**

**为什么这两件事不能互相替代。** 防御性公开的作用域是**新颖性**——它阻止他人就本公开的内容获得专利，这个作用域是**对世**的，与发布者日后是否实施无关。FTO 的作用域是**实施**——它判断按某个方案做产品时是否落入他人**已授权**的权利要求，这个作用域是**对实施者**的，与本公开无关。两者在法律上并不矛盾：**对方完全可以承认你的防御性公开有效，同时主张你实施的方案落入他的另一件专利。** 因此"我公开了，所以我不侵权"这个推理是错的，本文件不隐含这一推理。

**务实路径：先 knock-out，不做全量 FTO。** 完整 FTO 通常数万至十余万元、数周时间，在 PoC 阶段不现实。更有性价比的做法是先做一次**knock-out search**：

| 项 | 内容 |
|---|---|
| 目标 | 仅两件：`CN113316823A`、`CN121747954A` |
| 方法 | 逐条权利要求对照 LVMRP 设计，判断是否**每一项**特征都被满足 |
| 通过条件 | 若"患者级临床预测"这一特征不被满足 → knock-out 通过，可继续推进 |
| 未通过 | 回到设计层：改输出形式，或寻求许可 |
| 成本/周期 | 约数千至一两万元人民币，一周内出结论 |

这属于产品化前的必要支出，不是可选项。

另需说明：**"纵向 H&E + 分子轨迹"这个方向在本公开之前已见于公开文献**（见 [PRIOR_ART §4c](PRIOR_ART.md)）。本公开不对该方向主张任何优先权；这些文献反而压缩了他人就该宽泛方向圈占专利的空间。

**English**

This disclosure is a defensive publication. It is **not a freedom-to-operate (FTO) opinion**.

The **original framing** of LVMRP — longitudinal H&E → spatial Δ molecular map → treatment response — overlaps **substantially** with existing patents at the level of "longitudinal + molecular change + treatment response". The closest are `CN113316823A` (Roche / Pathomiq, and its EP family member `EP3857564A1`, **currently Pending**) and `CN121747954A`. Claim 1 of `CN113316823A` is in substance: molecular data + biopsy image data (including H&E) → machine-learning **clinical prediction** (platinum resistance; survival), and its specification at [0022] explicitly covers "patient treatment response analysis over time". **A defensive publication prevents others from patenting what you published; it does not prevent you from infringing an existing patent when you implement. Those are separate problems.**

The current design reduces the literal overlap by **limiting the output to spatial Δ maps and generating no patient-level clinical prediction** (see the negative limitation in [TECHNICAL_DISCLOSURE §1](TECHNICAL_DISCLOSURE.md#1-problem-definition--问题定义), the feature ordering in §5, and the claim-style boundary in §5.1). However:

- this is a boundary judgement based on **the claim language retrieved so far** — it is **not an FTO conclusion**, and it cannot guarantee that other patents will not apply;
- `CN113316823A` is **Pending**: its granted claims may differ from the published version;
- the claims of `CN121747954A` have not been compared line by line;
- whether "no patient-level output" holds up in practice as a genuine limitation, rather than being read as an *effective* treatment-response prediction (for instance when a user aggregates a map into a single readout and treats it as one), **is open to interpretation**.

**In short: if LVMRP is to be implemented rather than merely published defensively, a formal FTO analysis against the claims of `CN113316823A` should precede any development effort.**

> **This disclosure is a defensive publication and does not constitute an FTO opinion. "No patient-level prediction" is a product-design constraint, not a legal shield. Before productisation, a knock-out search or a full FTO analysis against `CN113316823A` and `CN121747954A` is required. A defensive publication prevents others from patenting what is disclosed here; it cannot substitute for an infringement assessment of already-granted patents.**

**Why the two cannot substitute for each other.** A defensive publication operates on **novelty** — it prevents others from patenting what is published here, and it does so *erga omnes*, independently of whether the publisher ever implements anything. FTO operates on **implementation** — it asks whether building a given design falls inside someone's **granted** claims, and it applies *to the implementer*, independently of what was published. The two are not in tension: **a third party can accept that your defensive publication is valid and still assert that your implementation falls inside a separate patent of theirs.** The inference "I published it, therefore I do not infringe" is wrong, and this document does not rely on it.

**A pragmatic route: knock-out first, full FTO later.** A complete FTO opinion typically costs tens of thousands of RMB and takes weeks, which is not realistic at PoC stage. A **knock-out search** is the cheaper first step:

| Item | Content |
|---|---|
| Targets | Two only: `CN113316823A`, `CN121747954A` |
| Method | Claim-by-claim comparison against the LVMRP design; test whether **every** feature is met |
| Pass condition | If the "patient-level clinical prediction" feature is not met → knock-out passes, proceed |
| Fail | Return to the design: change the output form, or seek a licence |
| Cost / time | Roughly a few thousand to ~20k RMB, conclusion within a week |

This is a necessary pre-productisation expense, not an optional one.

One further point: **the "longitudinal H&E + molecular trajectory" direction was already present in the public literature before this release** (see [PRIOR_ART §4c](PRIOR_ART.md)). This disclosure claims no priority over that direction; those publications weaken, rather than support, any attempt by anyone to claim the broad direction.

---

## 5. How to cite / 如何引用

Citation metadata lives in [CITATION.cff](CITATION.cff). GitHub renders a "Cite this repository" button from it.

引用元数据见 [CITATION.cff](CITATION.cff)；GitHub 会据此渲染 "Cite this repository" 按钮。

```bibtex
@misc{irixen2026lvmrp,
  title  = {LVMRP: Longitudinal Virtual Molecular Response Profiling},
  author = {{IRIXEN Patho Team}},
  year   = {2026},
  note   = {Defensive publication, v0.1.0},
  url    = {https://github.com/lanche-furry/LVMRP}
}
```

---

## 6. License / 许可

**中文**

- 代码与文档主体：**Apache License 2.0**（全文见 [LICENSE](LICENSE)）。
- 选它有一个特定理由：**第 3 条 Grant of Patent License**。任何使用者获得授权的同时，对本文公开内容作出专利主张的能力被进一步削弱——这与本仓库「防御性公开」的意图一致。
- 文字内容（说明性段落）也可在 **CC BY 4.0** 下复用；如两者冲突，以 Apache 2.0 为准。
- 归属与第三方组件声明见 [NOTICE](NOTICE)。

**English**

- Code and documentation: **Apache License 2.0** (full text in [LICENSE](LICENSE)).
- The choice is deliberate: **§3, Grant of Patent License**, further weakens the ability of any party to assert patent rights over the disclosed subject matter, which matches this repository's defensive-publication intent.
- Prose may additionally be reused under **CC BY 4.0**; where the two differ, Apache 2.0 governs.
- Attribution and third-party components: see [NOTICE](NOTICE).

---

## 7. Contributing / 贡献

**中文**

提交即表示你同意：(1) 你的贡献以 Apache License 2.0 授权；(2) 你**不保留**、也不会对已在本仓库公开的内容主张专利权。

**English**

By submitting a contribution you agree that (1) it is licensed under the Apache License 2.0, and (2) you retain **no** patent claim over the subject matter already disclosed in this repository.

---

## Repository layout / 仓库结构

| Path | Contents |
|---|---|
| [`README.md`](README.md) | This file / 本文件 |
| [`TECHNICAL_DISCLOSURE.md`](TECHNICAL_DISCLOSURE.md) | Enablement-level disclosure / 充分公开 |
| [`PRIOR_ART.md`](PRIOR_ART.md) | Boundary with prior art / 与现有技术的边界 |
| [`figures/architecture.md`](figures/architecture.md) | Mermaid architecture diagram / 架构图 |
| [`NOTICE`](NOTICE) | Attribution and defensive-publication notice |
| [`CITATION.cff`](CITATION.cff) | Citation metadata |
| [`LICENSE`](LICENSE) | Apache License 2.0, full text |

---

*Published 2026-10-18 by IRIXEN Patho Team. This document is a defensive publication: it exists to place the described subject matter in the public domain and to prevent any party from obtaining exclusive rights over it.*
