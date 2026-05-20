# ADR 0010 — SAM 2 zero-shot, hiera-tiny variant

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno
**Milestone:** M3
**Related spec:** design doc sec D1

## Context

M3 closes the pipeline with real segmentation. The master spec (sec M3) settled on
SAM 2 as the segmenter and deferred the zero-shot vs fine-tune decision to this ADR.

CarDD provides ~4k images with damage annotations in COCO format. SAM 2 was
trained on ~11M masks (the SA-V dataset). Three real options:

1. **Zero-shot, hiera-tiny** (~40 MB, ~1–2 s/mask on CPU).
2. **Zero-shot, hiera-base+** (~160 MB, ~3–4 s/mask on CPU).
3. **Fine-tune with CarDD.**

## Decision

**Zero-shot with `facebook/sam2-hiera-tiny`** as the default. A documented upgrade
path to `facebook/sam2-hiera-base-plus` via the `SEGMENTER_MODEL` env var.

## Consequences

### Positives

- Latency compatible with the M3 budget (~9–12 s for the full pipeline on CPU over
  3 typical damages). Hiera-base+ would reach 18–22 s and exceed `proxy_read_timeout`.
- No additional training pipeline. M5 trains the detector; the segmenter
  is not part of that cycle.
- Zero-shot quality sufficient for the demo. The mIoU differences between tiny and
  base+ are around 2–3 points on SA-V; with the Douglas–Peucker simplification
  applied in M3 (ADR 0012), the visible difference dilutes.

### Negatives

- Mask quality potentially inferior to a model fine-tuned with CarDD.
  Mitigation: the demo prioritizes a shape-complete pipeline over pixel-perfect masks;
  M5 can revisit this if a real insurtech requires higher fidelity.
- Inference costs scale linearly with the N damages. Mitigation: the detector already
  filters to COCO vehicles; a typical N is ≤5.

## Alternatives considered

### Fine-tune SAM 2 with CarDD

Rejected. CarDD has three orders of magnitude fewer masks than SA-V;
fine-tuning without regularization discipline significantly degrades
generalization. The behavior is documented in the SAM 1 literature
(catastrophic forgetting).

### SAM 1 (`facebook/sam-vit-base`)

Rejected. SAM 2 is strictly superior on SA-V eval; the `transformers`
API is equivalent. No technical reason to go back.

### Hiera-base+ as the default

Rejected on the latency budget (see the Consequences section). Available
as an opt-in via `SEGMENTER_MODEL=facebook/sam2-hiera-base-plus` for cases
where the operator accepts 2× latency.

## References

- Official SAM 2: <https://github.com/facebookresearch/sam2>
- Transformers Sam2Model: <https://huggingface.co/docs/transformers/main/en/model_doc/sam2>
- ADR 0001 — pretrained-first.
- ADR 0011 — severity-heuristic (downstream dependency).
- ADR 0012 — mask-encoding-polygon-cap.
