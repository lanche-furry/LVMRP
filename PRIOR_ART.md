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

**Representative / 代表：** `US20260024360`, `US20250342588A1`, `CN122454214A`, `CN113316823A`, `EP3857564A1`, `US20210249101A1`, SPARC (H&E → spatial gene-expression program activity maps)

**English**

- **What it covers.** Predicting spatial molecular content or quantitative protein abundance from a single H&E slide — the closest family on *output semantics*, since the deliverable is numeric rather than an image.
- **Where this proposal stops.**
  - The input requires **≥ 2 time points**; this family is single-time-point.
  - The target is **Δ**, not the absolute state; a static predictor can be a component, but is not the method.
  - No synthetic image is produced at any stage.

> SPARC infers **spatially-resolved gene-expression program activity maps** from H&E and associates them with treatment response. It is the nearest academic work on the *output object* — a spatial map rather than an image or a patch score. But its map is a **static** activity map, not a Δ between two time points, and it attaches a treatment-response association rather than withholding one. The gap this disclosure occupies is therefore narrow: static spatial maps on one side, clinical predictions on the other, and the Δ map in between.

> Work in this family that takes longitudinal H&E together with molecular data and predicts clinical response (resistance / survival) **outputs a clinical label**. This proposal outputs a spatially-resolved molecular change map Δ, and does **not** depend on paired H&E–IHC images for training.
>
> `CN113316823A` and its EP family member `EP3857564A1` are the most relevant instances: their pipeline runs H&E WSI → patch-level morphological score → **patient-level score** → treatment-outcome prediction. The distinguishing point is therefore **not** the longitudinal input but the **output level** — see §4b.
>
> `US20210249101A1` uses the phrase "molecular changes" for changes **relative to a reference genome** — gene mutation, copy-number variation, gene fusion, amplification, deletion — recovered in an **unsupervised** manner. That is a different technical meaning from Δ **between two time points of the same subject**, and this disclosure states the distinction explicitly rather than relying on the wording being read narrowly.

**中文**

- **覆盖内容。** 从单张 H&E 预测空间分子内容或定量蛋白丰度——在*输出语义*上最接近的一类，因为交付物是数值而非图像。
- **本方案止于何处。**
  - 输入要求**≥ 2 个时间点**；该类为单时点。
  - 目标是 **Δ**，不是绝对状态；静态预测器可以作为组件，但不是方法本身。
  - 任何阶段都不产生合成图像。

> SPARC 从 H&E 推断**空间分辨的基因表达程序活性图**，并将其与治疗响应关联。它是*输出对象*上最接近的学术工作——输出是空间图谱，而不是图像或 patch 分数。但其图谱是**静态**活性图，不是两个时间点之间的 Δ，而且它附带治疗响应关联，而非刻意不输出。因此本公开占据的缝隙很窄：一侧是静态空间图谱，另一侧是临床预测，中间才是 Δ 图谱。

> **该类工作从纵向 H&E + 分子数据预测临床反应（耐药/存活），输出是临床标签；本方案输出是空间分辨的分子变化图谱 Δ，且不依赖配对的 H&E-IHC 图像训练。**
>
> `CN113316823A` 及其 EP 同族 `EP3857564A1` 是其中最相关的实例：其流程为 H&E WSI → patch 级形态学分数 → **患者级分数** → 治疗结局预测。因此区别点**不在**纵向输入，而在**输出层级**——见 §4b。
>
> `US20210249101A1` 用 "molecular changes" 指**相对于参考基因组**的改变（突变、拷贝数变异、基因融合、扩增、缺失），且以**无监督**方式获得。这与**同一受试者两个时间点之间**的 Δ 是不同的技术含义；本公开显式写明这一区分，而不是依赖措辞被狭义解读。

---

## 4b. Closest prior art: longitudinal → clinical outcome prediction / 最接近的现有技术：纵向 → 临床疗效预测

**Representative / 代表：** `CN113316823A` (+ `EP3857564A1`, Roche / Pathomiq), `CN121747954A` (Guangzhou Medical University, 2026-03), `US2025299802A1` (Paige AI)

> **This is the closest prior art to this disclosure, and the boundary here is the sharpest in the document.** It is also the boundary most likely to be contested, because the overlap is real and lies in the *purpose* of the system, not in its plumbing.

**English**

`CN113316823A` (Roche / Pathomiq) is the most direct obstacle, and it is worth being precise about why. Its claim 1 is, in substance: receive a patient's **molecular data** → receive the patient's **biopsy image data, including H&E** → apply a machine-learning model to perform a **clinical prediction** — explicitly including whether the patient is *resistant or not resistant* to platinum-based therapy, and survival. Its specification, at paragraph [0022], is headed "patient treatment response analysis over time", so the **longitudinal** scenario is covered by the disclosure as well.

Four elements therefore coincide: longitudinal · H&E · molecular data · treatment-response prediction. **What does not coincide is the output level.** That is the only place where this disclosure separates itself, and it separates itself on purpose.

`CN113316823A` is currently **Pending**; its granted claims may differ from the published version, and that possibility is one reason the FTO caveat in [README §4b](README.md) matters.

| Reference | What it claims (per its claims as summarised to us) | Where this proposal stops |
|---|---|---|
| `CN113316823A` / `EP3857564A1` (Roche / Pathomiq, **Pending**) | Claim 1: molecular data + biopsy image data (incl. H&E) → ML **clinical prediction** (platinum resistance; survival). Spec [0022]: "patient treatment response analysis over time" | **No clinical prediction of any kind is produced.** No resistance call, no survival estimate, no patient-level score. Output terminates at the spatial Δ map and its uncertainty map. |
| `CN121747954A` | Pre/post-treatment **paired** slides → dynamic registration → biomarker expression change → spatial heterogeneity → fusion → **clinical efficacy prediction** | Input is **H&E**, not IHC; and the output stops at the molecular Δ map — it does not extend to an efficacy prediction. |
| `US2025299802A1` | Digital pathology image + treatment metadata → **treatment effectiveness assessment** | No effectiveness assessment or clinical decision output; the map is a quantitative object for the reader to interpret. |

The structural difference is deliberate and is stated as a negative limitation in the problem definition ([TECHNICAL_DISCLOSURE §1](TECHNICAL_DISCLOSURE.md#1-problem-definition--问题定义)): **the system does not output a patient-level treatment-response classification, a prognostic score, or a clinical decision recommendation.**

**中文**

`CN113316823A`（罗氏 / Pathomiq）是最直接的障碍，值得说清为什么。其权利要求 1 的实质是：接收患者的**分子数据** → 接收患者的**活检图像数据（含 H&E）** → 用机器学习模型进行**临床预测**——明确包括"对铂类药物治疗是耐药还是不耐药"，以及存活率。其说明书 [0022] 节标题为"随时间的患者治疗反应分析"，因此**纵向**场景也在其公开范围内。

于是四个要素全部命中：纵向 · H&E · 分子数据 · 治疗反应预测。**不重合的是输出层级。** 这是本公开唯一能切割的地方，也是刻意切割的地方。

`CN113316823A` 当前状态为 **Pending**，最终授权权利要求可能与公开版本不同——这也是 [README §4b](README.md) 那段 FTO 提示为什么重要。

| 文献 | 其权利要求所述内容（按其要点转述） | 本方案止于何处 |
|---|---|---|
| `CN113316823A` / `EP3857564A1`（罗氏 / Pathomiq，**Pending**） | 权利要求 1：分子数据 + 活检图像数据（含 H&E）→ 机器学习**临床预测**（铂类耐药；存活率）。说明书 [0022]："随时间的患者治疗反应分析" | **不产生任何形式的临床预测。** 不做耐药判断、不给生存估计、不产生患者级分数。输出终止于空间 Δ 图谱与其不确定性图。 |
| `CN121747954A` | 治疗前后**配对**切片 → 动态配准 → 生物标志物表达变化 → 空间异质性 → 融合 → **临床疗效预测** | 输入是 **H&E**，不是 IHC；且输出停在分子 Δ 图谱，不延伸到疗效预测。 |
| `US2025299802A1` | 数字病理图像 + 治疗元数据 → **治疗有效性评估** | 不输出有效性评估或临床决策；图谱是供读者解读的定量对象。 |

这一结构差异是刻意设置的，并已在问题定义中以**否定性限定**写明（见 [TECHNICAL_DISCLOSURE §1](TECHNICAL_DISCLOSURE.md#1-problem-definition--问题定义)）：**系统不输出患者级别的治疗反应分类、预后评分或临床决策建议。**

---

## 4c. The longitudinal direction was already named in the literature / "纵向"方向已在文献中出现

**English**

This direction is **not** this disclosure's invention, and this document says so plainly rather than relying on a search engine's silence.

Two publications predate the release of this document (2026-10-18):

| Date | What it established |
|---|---|
| 2026-09-18 | A review names **"longitudinal morphomolecular trajectory prediction"** explicitly as a future direction. |
| 2026-10-15 | A JoVE paper demonstrates **"Serial H&E Pathology"** as a **longitudinal digital readout**, based on nuclear-morphology change between time points. |

Both precede this release. **This disclosure therefore claims no priority over the longitudinal direction, and none over the idea of reading change over time from H&E.** What it publishes is a specific combination — longitudinal H&E input, a spatially-resolved Δ target, and an output that stops below clinical prediction — not the direction itself.

These are papers, not patent claims, so they do not affect freedom to operate directly. Their effect is the opposite of a threat: **they are prior art against anyone who would later try to claim the broad direction**, which narrows the space available for exclusivity over "longitudinal H&E molecular prediction" generally. That is a public good, and this document is glad to point at it.

> Citation note: the two references above are stated as they were surfaced in a private prior-art review on 2026-10-18, with dates and technical content but without bibliographic identifiers. **Full author, title and DOI should be completed from the reviewer's records before this boundary is relied upon in any formal proceeding.** Stating a reference without a complete citation is weaker than stating one with it; the gap is recorded here rather than papered over.

**中文**

这个方向**不是**本公开的发明，本文直说，而不是依赖"检索没搜到"。

有两篇文献早于本文档的发布日（2026-10-18）：

| 日期 | 它确立了什么 |
|---|---|
| 2026-09-18 | 一篇综述明确把 **"longitudinal morphomolecular trajectory prediction"** 列为未来方向。 |
| 2026-10-15 | 一篇 JoVE 论文展示了 **"Serial H&E Pathology"** 作为**纵向数字读出**的可行性，基于时间点之间的核形态变化。 |

两篇都早于本次发布。**因此本公开不对"纵向"方向主张任何优先权，也不对"从 H&E 读取随时间变化"这一想法主张优先权。** 它公开的是一个特定组合——纵向 H&E 输入、空间分辨的 Δ 目标、以及止于临床预测之前的输出——而不是方向本身。

这些是论文，不是专利权利要求，因此不直接影响自由实施。它们的作用与威胁相反：**它们是针对任何日后试图圈占该宽泛方向者的现有技术**，从而压缩了"纵向 H&E 分子预测"这一泛方向上的排他空间。这对公共知识是好事，本文件乐于指出。

> 引用说明：上述两条按 2026-10-18 一次内部现有技术核查中所见记录（有日期与技术内容，无书目标识符）。**在任何正式程序依赖本边界之前，应从核查者的记录中补全作者、标题与 DOI。** 有完整引文的引用强于无引文的引用；这里的缺口被记录下来，而不是被掩盖。

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

Taken across all six families, plus the closest-art group in §4b: **every one of them lacks at least one of the three axes.**

| Family | A. Longitudinal input | B. Delta target | C. Non-image output |
|---|:---:|:---:|:---:|
| 1. H&E → virtual IHC images | ✗ | ✗ | ✗ |
| 2. Unpaired virtual staining | ✗ | ✗ | ✗ |
| 3. Label-free → virtual staining | ✗ | ✗ | ✗ |
| 4. H&E → spatial molecular | ✗ | ✗ | ✓ |
| 4b. Longitudinal → clinical outcome (`CN113316823A`, `CN121747954A`, `US2025299802A1`) | ✓ | ✗ (clinical label) | ✓ |
| 5. Serial-section (Z-axis) | ✗ (depth ≠ time) | ✗ | ✗ |
| 6. Longitudinal radiology | ✓ | ✗ (response label) | ✓ |

**This document claims no priority and no originality.** It does not assert that any element above is new, nor that the combination has never been described; §4b exists precisely because the closest art overlaps the original framing substantially, and §4c records two publications that named the longitudinal direction before this release. What the disclosure does is publish the specific combination — longitudinal H&E input, a spatially-resolved delta target, and a non-image output that stops short of clinical prediction — so that it enters the public domain and no party can obtain exclusivity over it.

A reader who finds prior art holding all three axes should read that as this boundary being wrong, and is invited to say so. Correcting the boundary is more useful to everyone than defending it.

**中文**

综观六类，加上 §4b 的最接近现有技术：**每一类都至少缺少三个轴中的一个。**

| 类别 | A. 纵向输入 | B. Δ 目标 | C. 非图像输出 |
|---|:---:|:---:|:---:|
| 1. H&E → 虚拟 IHC 图像 | ✗ | ✗ | ✗ |
| 2. 非配对虚拟染色 | ✗ | ✗ | ✗ |
| 3. 无标记 → 虚拟染色 | ✗ | ✗ | ✗ |
| 4. H&E → 空间分子 | ✗ | ✗ | ✓ |
| 4b. 纵向 → 临床疗效（`CN113316823A`、`CN121747954A`、`US2025299802A1`） | ✓ | ✗（临床标签） | ✓ |
| 5. 序列切片（Z 轴） | ✗（深度 ≠ 时间） | ✗ | ✗ |
| 6. 纵向放射影像 | ✓ | ✗（响应标签） | ✓ |

**本文件不主张任何优先权，也不主张任何独创性。** 它既不断言上述任一要素是新的，也不断言该组合从未被描述过；§4b 之所以存在，正是因为最接近的现有技术与原始定位有实质重叠。本公开所做的是把**这个特定组合**——纵向 H&E 输入、空间分辨的变化量目标、以及止于临床预测之前的非图像输出——公开置入公共领域，使任何主体都无法对其取得排他权。

若读者发现同时具备三个轴的现有技术，应视为本边界判断有误，并欢迎指出。**修正边界对所有人都比维护边界更有用。**

---

*This document is a defensive publication. Where a boundary is stated, it is stated to be auditable — not to imply the prior art is narrow.*
