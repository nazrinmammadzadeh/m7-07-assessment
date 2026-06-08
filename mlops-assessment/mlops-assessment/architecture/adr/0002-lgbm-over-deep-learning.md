# ADR 0002 — Use LightGBM over LSTM / Transformer for Failure Prediction

**Date:** 2026-06-07  
**Status:** Accepted  
**Deciders:** ML Lead, Data Science Team  
**Review trigger:** Quarterly model review; revisit if production AUC falls below 0.80

---

## Context

The failure prediction task takes a 168-step time-series of vibration and temperature per sensor and outputs `failure_prob_72h` — the probability that the sensor will fail within the next 72 hours. The prediction must be served via a synchronous HTTP endpoint with p95 latency ≤ 120 ms, without a GPU, on `c6i.large` instances (2 vCPU, 4 GB RAM).

Three model families were evaluated on a held-out test set of 18 months of sensor data from a comparable factory installation:

| Model | AUC-ROC | Precision@0.5 | p95 CPU inference | Model size | GPU needed |
|---|---|---|---|---|---|
| **LightGBM** (hand-crafted rolling stats + FFT features) | **0.86** | **0.83** | **≤ 5 ms** | **45 MB** | **No** |
| LSTM (2 layers, hidden=128) | 0.88 | 0.85 | ~80 ms | 120 MB | Borderline |
| Temporal Fusion Transformer | 0.91 | 0.88 | ~200 ms | 410 MB | Yes (practical) |

---

## Decision

**LightGBM with hand-crafted rolling statistics and FFT features.**

---

## Reasoning

### Latency budget is the hard constraint

The HTTP SLO is p95 ≤ 120 ms. After accounting for network (≤ 5 ms), Redis feature lookup (≤ 5 ms), and serialization (≤ 3 ms), the model has ≤ 107 ms of budget. LightGBM clears this comfortably (≤ 5 ms p95). LSTM is borderline at ~80 ms p95 and requires batching tricks that complicate the single-sensor synchronous API contract. TFT at ~200 ms on CPU violates the SLO outright; it would require GPU nodes, tripling the monthly compute cost.

### Model size enables baked-in images

At 45 MB, the LightGBM artefact can be baked into the container image without making images unwieldy. This is the key enabling decision for the rollback strategy: rolling back the model is identical to rolling back the image (a single `kubectl set image` command). A 410 MB TFT artefact would make per-version images impractical and force a runtime-mount strategy, which decouples image and model versions and complicates rollback (see `runbooks/rollback.md`).

### Interpretability builds operator trust

LightGBM provides native SHAP feature importances. Maintenance schedulers can see explanations such as "vibration RMS over the last 24 h was 3.2σ above the 7-day baseline" alongside each prediction. LSTM and TFT outputs are opaque to non-ML staff and require separate explainability libraries that add latency and maintenance burden.

### AUC trade-off is acceptable

LightGBM achieves AUC 0.86 vs. TFT's 0.91. The 5-point AUC difference translates to approximately 4–5 additional missed failures per 1,000 at-risk sensors per month. Given that the system supplements (not replaces) scheduled maintenance, this is an acceptable operational trade-off. The AUC gate in the model registry (`≥ 0.82`) ensures a future model regression is caught before production.

---

## Consequences

### Positive

- p95 inference latency of ≤ 5 ms leaves 115 ms of headroom for network and feature lookup.
- 45 MB artefact baked into image; rollback = image rollback (one command).
- SHAP explanations available without additional infrastructure.
- Training time < 5 minutes on a single `c6i.2xlarge`; fast retraining cycle.

### Negative / Risks

- LightGBM cannot learn arbitrary long-range temporal dependencies from raw sequences. FFT peaks and rolling statistics are an imperfect proxy for complex vibration patterns.
- If the sensor fleet is expanded with new sensor types exhibiting different failure signatures, the feature engineering step must be revisited.
- If quarterly review shows AUC < 0.80 in production (via proxy labels from confirmed maintenance logs), the team should re-evaluate TFT with GPU nodes (`g5.xlarge`, ≈ USD 250/month per node).

### Review schedule

- **Quarterly**: compare model AUC on the latest 90-day held-out set. If AUC < 0.82, trigger a retrain. If retrained LightGBM cannot reach 0.82, open a model-family review issue.
- **On significant fleet change**: any addition of > 5,000 new sensor types or a factory reconfiguration triggers an ad-hoc model review.
