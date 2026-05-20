# ADR 0017 — Result of the RT-DETRv2 fine-tune on CarDD: deploy

**Status:** Accepted
**Date:** 2026-05-06
**Author:** Jesús Moreno

## Context

[ADR 0001](0001-pretrained-first.md) established in M0 that the pipeline runs on pretrained weights until M5, where RT-DETRv2 is fine-tuned on CarDD and the decision is made on whether the custom weights replace the pretrained ones in production. The M5 spec closed with D7: thresholds **pre-committed before seeing any numbers** so that the verdict is defensible regardless of the outcome:

| Result | Action |
|---|---|
| mAP@0.5 ≥ 0.40 ∧ latency ≤ 1.2× baseline | **`deploy`** |
| 0.20 ≤ mAP@0.5 < 0.40 | `publish_no_deploy` (keep pretrained-mock) |
| mAP@0.5 < 0.20 ∨ latency > 1.2× | `publish_with_diagnosis` (keep pretrained-mock + diagnose) |

This ADR documents the actual result of the experiment and the resulting decision.

## Decision

**Verdict: `deploy`.** The `models/rtdetr-cardd.pt` weights replace the pretrained `rtdetr-l.pt` in production via the `DETECTOR_WEIGHTS` env var configured in Coolify (T9, sec D6 of the M5 spec).

Eval metrics ([raw JSON](../metrics/baseline-vs-finetuned.json), [narrative](../metrics/baseline-vs-finetuned.md)):

| D7 criterion | Threshold | Actual | Passes |
|---|---|---|---|
| mAP@0.5 (test split) | ≥ 0.40 | **0.701** | ✓ |
| Latency ratio (CPU warm) | ≤ 1.2× | **1.057×** | ✓ |

Per-class mAP@0.5: `broken_glass` 0.966, `broken_lamp` 0.819, `flat_tire` 0.799, `dent` 0.591, `crack` 0.523, `scratch` 0.509. Detail in [`baseline-vs-finetuned.md`](../metrics/baseline-vs-finetuned.md#per-class).

## Consequences

### Positives

- **Real domain categories.** The classes predicted by the detector are now those of the `DamageType` enum (`dent`, `scratch`, etc.) — not COCO proxies translated by `coco_to_damage_type` (the M2 placeholder). [ADR 0008](0008-rtdetr-vs-yolo.md) anticipated this replacement; D8 of the M5 spec left the detector ready to serve it with no further runtime changes (commit [`2f8c8b8`](https://github.com/Jesusmcyc/car-crash-analyzer/commit/2f8c8b8)).
- **Aggregate mAP in production-grade territory.** 0.701 clears the 0.40 floor that the spec defined as "feels useful in a demo" by a wide margin. The probability that a random demo image shows correct detections is high.
- **Latency preserved.** The 32 ms delta (5.7%) is negligible for the demo target (~3 s warm). The backbone is the same as the pretrained one; only the weights change.
- **The project roadmap was fulfilled.** [ADR 0001](0001-pretrained-first.md) promised "the obvious fallback: if the fine-tune does not improve the pretrained baseline, the system works just the same with the original weights". The fine-tune did improve, so it ships — but the rollback documented in spec D6 (delete the env var, redeploy, fall back to pretrained-mock) remains available if something is discovered post-deploy.

### Negatives

- **Per-class gap between edge-bound classes and texture-bound classes.** `broken_glass`/`broken_lamp`/`flat_tire` sit at mAP ≥ 0.80; `dent`/`crack`/`scratch` at 0.50-0.59. This does not invalidate the verdict — the aggregate clears the threshold and the behavior is consistent with the nature of the damage — but the demo presents three implicit confidence tiers that the frontend should surface (a caveat to the user when `confidence < 0.6`).
- **`flat_tire` 0.799 has high statistical variance.** Only 32 instances in the test split (the class with the fewest samples). The informal confidence interval is around ±0.10 — a different random test split could show mAP of 0.65-0.85. The demo narrative should communicate "≥ 0.70 on test" rather than citing 0.799 as a definitive number. Detail in [`baseline-vs-finetuned.md` sec Per-class](../metrics/baseline-vs-finetuned.md#per-class).
- **We did not measure overfitting quantitatively.** The report only includes the test split. The val split (810 imgs, mAP@0.5 = 0.700 according to the last training epoch) is implicit in the Ultralytics plots inside `rtdetr_cardd_run1/` on Drive. The fact that test (0.701) and val (0.700) are practically identical suggests good generalization, but it was not committed as a formal metric.
- **Real training wall-clock underestimated by the spec.** Spec D3 estimated "~2-3 h on a Colab T4". The pure `model.train()` time was **1.234 h (74 min)** according to `results.csv`, but the total wall-clock with setup overhead × 2 sessions (clone, pip, drive mount, label scan) and the disconnect gap landed at **~2-3 h**. The second attempt required paying for Pay As You Go ($10 USD = 100 compute units, 8 consumed) to unblock the quota.
### Neutral / follow-ups

- **Class weights not applied** (a D3-deferred decision in the spec). The next iteration, M5.5, could add them if `scratch`/`crack` turn out to be frequent in real usage and their mAPs limit usefulness. Does not block T9.
- **`APP_VERSION` in Coolify still not injected** (inherited from M4). The T9 post-deploy smoke test accepts `model_version: "rtdetr-cardd-unknown"` as a provisional green per the spec clause (sec 348). A 1-line fix in the Coolify UI; does not block.

## Alternatives considered

1. **Keep pretrained-mock COCO + `coco_to_damage_type` (verdict `publish_no_deploy`).** Rejected — the D7 threshold (mAP@0.5 ≥ 0.40) was cleared comfortably. Keeping pretrained would be honesty about a false data point (pretrained-mock does not report its real mAP because it has no CarDD classes; D7 already covers that argument).
2. **Wait for a second run with class weights before deploying.** Rejected — the current verdict already meets the deploy criteria. A run with class weights is valuable as M5.5, but delaying the deployment of the already-trained weights by another 1-2 days adds nothing — the current weights generate demo value from the moment they land in Coolify.
3. **Train for more epochs (extend `epochs=50` to 100, `patience=20`).** Rejected — Ultralytics triggered early-stop at epoch 40 with 10 epochs of no improvement since epoch 30. More epochs would have been wasted time and compute. The best.pt from epoch 30 captures the peak.
4. **A larger backbone (RT-DETR-x instead of -l).** Rejected implicitly by D3 of the spec — the CPU-only demo of the production runtime is already close to the ceiling of tolerable latency with -l. -x would add ~50% latency for an estimated improvement on the order of mAP +0.02-0.05; a trade-off not justified for a demo.

## References

- [ADR 0001](0001-pretrained-first.md) — pretrained-first; this ADR closes that promise.
- [ADR 0008](0008-rtdetr-vs-yolo.md) — COCO→DamageType placeholder; obsolete in use, but `coco_to_damage_type` is kept as a fallback (D8 of the M5 spec).
- M5 spec — D3 (hyperparams), D4 (eval), D6 (deploy), D7 (verdict thresholds).
- [`Docs/metrics/baseline-vs-finetuned.json`](../metrics/baseline-vs-finetuned.json) — raw numbers.
- [`Docs/metrics/baseline-vs-finetuned.md`](../metrics/baseline-vs-finetuned.md) — per-class interpretation and operational deviations.
