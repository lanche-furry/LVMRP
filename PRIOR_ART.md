# Prior Art and Boundary — LVMRP

# 现有技术与边界 — LVMRP

**Version:** v0.1.0 · **Release date:** 2026-10-18 · **Author:** IRIXEN Patho Team
**License:** Apache License 2.0 · **Nature:** defensive publication

> **How to read this document.** Each entry names a family of prior work and states, as plainly as possible, **where this proposal stops**. The purpose is not to claim the prior work is narrow, and not to claim this proposal is novel over all of it — it is to make the boundary auditable, so that a later reader can see exactly which combination this disclosure puts in the public domain.
>
> **阅读方式。** 每条列出一类现有技术，并尽量直白地说明**本方案止于何处**。目的不是论证现有技术范围窄，也不是宣称本方案相对它们全部具备新颖性，而是让边界可被审计——后来的读者能看清本公开究竟把哪一个组合置入了公共领域。

**Three axes used throughout / 全文使用的三个轴：**

| Axis | 本方案 | Meaning |
|---|---|---|
| A. Longitudinal input | ≥ 2 time points, same subject | 输入是否为纵向 |
| B. Delta target | predicts Δ, not the state | 预测目标是否为变化量 |
| C. Non-image output | maps, never RGB | 输出是否为非图像 |

---

## 1. H&E → virtual IHC **images** / H&E → 虚拟 IHC **图像**

**Representative / 代表：** `CN111819569B`, `US-12367691-B2`

**English**

- **What it covers.** Generating an immunohistochemistry image from a hematoxylin-eosin image: the deliverable is a rendered image that stands in for a real IHC stain, and the training signal is built from H&E–IHC relationships.
- **Where this proposal stops.**
  - It does **not** generate an RGB stained image — the output is a numeric delta map plus an uncertainty map.
  - It does **not** require paired H&E–IHC training data; it requires same-subject longitudinal H&E with molecular measurements.
  - It predicts **Δ**, not an image, and not a static level.

**中文**

- **覆盖内容。** 由 H&E 图像生成免疫组化图像：交付物是一张替代真实 IHC 染色的渲染图像，训练信号建立在 H&E–IHC 关系之上。
- **本方案止于何处。**
  - **不**生成 RGB 染色图像，输出是数值型变化图谱 + 不确定性图；
  - **不**依赖配对的 H&E–IHC 训练数据，而依赖同一受试者的纵向 H&E 与分子测量；
  - 预测的是 **Δ**，不是图像，也不是静态水平。

---

## 2. **Unpaired** H&E/IHC virtual staining / **非配对** H&E/IHC 虚拟染色

**Representative / 代表：** `WO2025168731A1`, confusion-GAN

**English**

- **What it covers.** Translating between H&E and IHC **domains** without paired exemplars, typically with a cycle-consistency or confusion-based adversarial objective. The aim is still an image in the target stain.
- **Where this proposal stops.**
  - The output is **not an image**.
  - Training uses **same-subject longitudinal pairs**, not cross-subject unpaired domain translation.
  - The objective is a molecular change, not a stain appearance.

**中文**

- **覆盖内容。** 在无配对样本的情况下做 H&E 与 IHC 的**域间转换**，通常使用循环一致性或基于混淆的对抗目标。目标仍是目标染色下的图像。
- **本方案止于何处。**
  - 输出**不是图像**；
  - 训练使用**同一受试者的纵向配对**，而非跨受试者的非配对域转换；
  - 目标是分子变化，而非染色外观。

---

## 3. Label-free → virtual staining / 无标记 → 虚拟染色

**Representative / 代表：** UC Regents family (`CN114945954B`), `US20250259462A1` (autofluorescence / FLIM)

**English**

- **What it covers.** Recovering a stained appearance from a **label-free** modality — autofluorescence, FLIM, or other physical measurements — instead of from an already-stained slide.
- **Where this proposal stops.**
  - The input is **H&E**, not a label-free modality.
  - The output is **Δ**, not an image.
  - This proposal asserts **no rights over the label-free imaging path**; the two are disjoint on both input and output.

**中文**

- **覆盖内容。** 从**无标记**模态（自发荧光、FLIM 或其他物理测量）恢复染色外观，而不是从已染色切片出发。
- **本方案止于何处。**
  - 输入是 **H&E**，不是无标记模态；
  - 输出是 **Δ**，不是图像；
  - 本方案**不主张对无标记成像路径的任何权利**；两者在输入与输出上都互不相交。

---

## 4. H&E → spatial molecular / quantitative protein / H&E → 空间分子 / 定量蛋白

**Representative / 代表：** `US20260024360`, `US20250342588A1`, `CN122454214A`

**English**

- **What it covers.** Predicting spatial molecular content or quantitative protein abundance from a single H&E slide — the closest family on *output semantics*, since the deliverable is numeric rather than an image.
- **Where this proposal stops.**
  - The input requires **≥ 2 time points**; this family is single-time-point.
  - The target is **Δ**, not the absolute state; a static predictor can be a component, but is not the method.
  - No synthetic image is produced at any stage.

> This is the family the proposal is nearest to, and therefore the one where the boundary must be sharpest: **same output semantics, different input shape and different target.**

**中文**

- **覆盖内容。** 从单张 H&E 预测空间分子内容或定量蛋白丰度——在*输出语义*上最接近的一类，因为交付物是数值而非图像。
- **本方案止于何处。**
  - 输入要求**≥ 2 个时间点**；该类为单时点。
  - 目标是 **Δ**，不是绝对状态；静态预测器可以作为组件，但不是方法本身。
  - 任何阶段都不产生合成图像。

> 本方案与这一类最近，因此边界必须最锐利：**相同的输出语义，不同的输入形态与不同的预测目标。**

---

## 5. Serial-section (Z-axis) → virtual IHC / 序列切片（Z 轴）→ 虚拟 IHC

**Representative / 代表：** `CN122090442A`

**English**

- **What it covers.** Using **serial sections through one block** — the Z axis — to relate an H&E section to an IHC section, so that a depth-adjacent slice supplies the supervision.
- **Where this proposal stops.**
  - The time axis is **real biological time** (pre/post treatment), not physical depth through a block.
  - The output is **not an image**.
  - The scientific question is treatment response, which a Z-axis method cannot address by construction.

**中文**

- **覆盖内容。** 用**同一蜡块的连续切片**（Z 轴）把 H&E 切片与 IHC 切片关联，以相邻层的切片提供监督。
- **本方案止于何处。**
  - 时间轴是**真实生物学时间**（治疗前/后），不是穿过蜡块的物理深度；
  - 输出**不是图像**；
  - 科学问题是治疗响应，Z 轴方法在构造上就无法回答。

---

## 6. Longitudinal imaging (non-pathology) / 纵向影像组学（非病理）

**Representative / 代表：** CT/MRI longitudinal response prediction

**English**

- **What it covers.** Predicting treatment response from **longitudinal radiological** imaging — the same *longitudinal* idea, applied to CT/MRI rather than to whole-slide pathology.
- **Where this proposal stops.**
  - The input is **pathology WSI**, not radiology volumes.
  - The output is **spatial molecular Δ**, not a response label or a radiological change.
  - This proposal asserts **no rights over the radiological imaging domain**.

**中文**

- **覆盖内容。** 从**纵向放射影像**预测治疗响应——同样的*纵向*思想，作用于 CT/MRI 而非全切片病理。
- **本方案止于何处。**
  - 输入是**病理 WSI**，不是放射影像体数据；
  - 输出是**空间分子 Δ**，不是响应标签或影像学变化；
  - 本方案**不主张对放射影像领域的任何权利**。

---

## Summary / 小结

**English**

Taken across all six families: **every one of them lacks at least one of the three axes.**

| Family | A. Longitudinal input | B. Delta target | C. Non-image output |
|---|:---:|:---:|:---:|
| 1. H&E → virtual IHC images | ✗ | ✗ | ✗ |
| 2. Unpaired virtual staining | ✗ | ✗ | ✗ |
| 3. Label-free → virtual staining | ✗ | ✗ | ✗ |
| 4. H&E → spatial molecular | ✗ | ✗ | ✓ |
| 5. Serial-section (Z-axis) | ✗ (depth ≠ time) | ✗ | ✗ |
| 6. Longitudinal radiology | ✓ | ✗ (response label) | ✓ |

No family holds all three. **That specific combination — longitudinal H&E input, a spatially-resolved delta target, and a non-image output — is what this disclosure publishes.**

This table is an argument about combination, not about any single axis being unknown. A reader who finds prior art holding all three should read that as this boundary being wrong, and is invited to say so.

**中文**

综观六类：**每一类都至少缺少三个轴中的一个。**

| 类别 | A. 纵向输入 | B. Δ 目标 | C. 非图像输出 |
|---|:---:|:---:|:---:|
| 1. H&E → 虚拟 IHC 图像 | ✗ | ✗ | ✗ |
| 2. 非配对虚拟染色 | ✗ | ✗ | ✗ |
| 3. 无标记 → 虚拟染色 | ✗ | ✗ | ✗ |
| 4. H&E → 空间分子 | ✗ | ✗ | ✓ |
| 5. 序列切片（Z 轴） | ✗（深度 ≠ 时间） | ✗ | ✗ |
| 6. 纵向放射影像 | ✓ | ✗（响应标签） | ✓ |

没有任何一类同时具备三项。**本公开置入公共领域的，正是这个特定组合——纵向 H&E 输入、空间分辨的变化量目标、非图像输出。**

这张表论证的是**组合**，而不是说任一单轴未知。若读者发现了同时具备三项的现有技术，应视为本边界判断有误，并欢迎指出。

---

*This document is a defensive publication. Where a boundary is stated, it is stated to be auditable — not to imply the prior art is narrow.*
