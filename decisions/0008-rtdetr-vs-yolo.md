# ADR 0008 — Default detector: RT-DETRv2 over YOLOv8

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

M2 introduces the first real model into the pipeline, replacing the deterministic mock from M1. Two pretrained candidates via Ultralytics: RT-DETRv2 (a DETR-style transformer) and YOLOv8 (a one-stage CNN). Both solve general detection; both come trained on COCO; both expose an identical API in Ultralytics.

The question is not "which is better in the abstract" — it is **which one best serves the demo's use case (CarDD, ~4k images, overlapping damage, CPU)**.

## Decision

**Default: RT-DETRv2** (weights `rtdetr-l.pt`). **Functional fallback: YOLOv8** (weights `yolov8s.pt`). Switching via the `DETECTOR_BACKEND ∈ {"rtdetr", "yolov8"}` env var, with no recompilation.

COCO → DamageType mapping: **an explicit placeholder until M5**. Pretrained COCO does not know `dent`/`scratch`/etc., so M2 detects vehicles (classes `car`, `truck`, `bus`, `motorcycle`) and assigns them a cyclic DamageType. M5 fine-tunes on CarDD, and the custom weights return real classes from the enum.

## Consequences

### Positives

- **Set prediction without NMS.** Vehicle damage overlaps in CarDD (a scratch over a dent, broken glass adjacent to a bent frame). RT-DETR does not collapse nearby detections the way YOLOv8 with NMS tends to. This advantage materializes once M5's custom weights enter the pipeline.
- **Sample efficiency.** CarDD has ~4k images — small. DETRs learn better on datasets of this size than YOLO trained from scratch.
- **Unified Ultralytics API.** The cost of keeping YOLOv8 as a fallback is practically zero — the same `Model(image)` → `Result.boxes` signature. The `get_detector(settings)` factory selects the backend, and the whole pipeline consumes it opaquely.
- **Technical honesty.** The COCO→DamageType placeholder is documented prominently (this ADR plus the module). The demo does not lie: M2 validates the pipeline plumbing; M5 makes it a real damage detector.

### Negatives

- **M2 detections are not damage-specific.** The production demo after M2 shows "this pipeline detects vehicles in images" more than "this pipeline detects vehicle damage". Mitigated by M5 (running in parallel), which replaces the weights via env var without touching code.
- **RT-DETR-l is ~70 MB vs YOLOv8s ~22 MB.** The cold start of the VPS's first boot is 3× longer if it starts with RT-DETR. Mitigated by volume caching (ADR 0009).
- **Higher CPU inference latency with RT-DETR.** Expected 5–8s vs 2–3s for YOLOv8. Acceptable for a demo (no production SLO); the YOLOv8 fallback is available if M5 shows the upgrade does not justify the latency.

## Alternatives considered

1. **YOLOv8 as the default.** Faster on CPU, lighter download. Rejected because the problem (overlapping damage) is exactly where RT-DETR shines — fixing the default based on latency alone would sacrifice the architectural advantage.
2. **Keeping the mock until M5.** Without a real pipeline, the M1 deploy never validates that the Docker container can handle PyTorch + Ultralytics + weight caching. Risk of discovering the problem at the end of the project. Rejected.
3. **Pretrained weights on a closer dataset (Open Images, not COCO).** Open Images has no vehicle-damage classes either. Same placeholder problem; no upside. Rejected.

## References

- Master spec: design doc sec M2 and sec 4 (ADRs).
- M2 design decisions: design doc sec 2 D2 + D3.
- ADR 0001 — pretrained-first: [`0001-pretrained-first.md`](0001-pretrained-first.md).
- Implementation: `backend/app/pipeline/detector.py`.
