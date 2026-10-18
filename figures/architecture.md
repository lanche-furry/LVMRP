# Architecture — LVMRP

```mermaid
flowchart TB
    subgraph IN["Inputs"]
        A0["H&E whole-slide image<br/>t0"]
        A1["H&E whole-slide image<br/>t1"]
        C["Clinical covariates c<br/>treatment · interval"]
    end

    subgraph ENC["Frozen encoder (no gradients)"]
        E0["Pathology foundation model<br/>H-Optimus-0 · ViT-Giant · patch 14<br/>tokens [N, 261, 1536]"]
        E1["Pathology foundation model<br/>H-Optimus-0 · ViT-Giant · patch 14<br/>tokens [N, 261, 1536]"]
        P0["Attention pool"]
        P1["Attention pool"]
    end

    subgraph DEC["Response decoder (trainable, 10–50 M)"]
        Z0["z0"]
        Z1["z1"]
        F["Condition fusion MLP"]
        G["γ (condition vector)"]
        S["SPADE spatial modulation<br/>multi-resolution γ(h,w) / β(h,w)"]
        DH["Δ head"]
        UH["Uncertainty head"]
    end

    subgraph OUT["Outputs (never RGB images)"]
        DM["Δ molecular map<br/>spatially resolved change"]
        UM["Uncertainty map<br/>per-region variance"]
    end

    A0 --> E0 --> P0 --> Z0
    A1 --> E1 --> P1 --> Z1
    Z0 --> F
    Z1 --> F
    C --> F
    F --> G --> S
    S --> DH --> DM
    S --> UH --> UM

    classDef frozen fill:#eef3fb,stroke:#0b3d91,stroke-width:1px,color:#0f1720
    classDef train fill:#fff7ea,stroke:#8a5a00,stroke-width:1px,color:#0f1720
    classDef output fill:#eefaf1,stroke:#2e7d4f,stroke-width:1px,color:#0f1720
    class E0,E1,P0,P1 frozen
    class F,S,DH,UH,G train
    class DM,UM output
```

The pathology foundation model is the **worked embodiment** (H-Optimus-0, ViT-Giant, patch 14, 224 × 224 input, token tensor `[N, 261, 1536]`). It is frozen end to end and is **replaceable** — UNI, Virchow or Prov-GigaPath are expected to work with the same downstream design, subject to matching the token dimension.

The condition is injected through **multi-resolution SPADE spatial modulation**, not through a global FiLM vector. The reason is the failure mode recorded in [README §4](../README.md#4-current-evidence--honest-disclosure): condition sensitivity L1 of 0.0112 in the static route, i.e. a condition signal that is averaged away.

Nothing in this diagram produces an RGB stained image. The output is two maps: a quantitative delta map and an uncertainty map.
