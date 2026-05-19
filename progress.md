# Progress

A milestone-by-milestone narrative of how this project was built. Roughly one screen per milestone.

## M0 / M1 — Pretrained skeleton

The project opened with a working end-to-end skeleton before training anything: a FastAPI backend with RT-DETRv2 and SAM 2 wired in via Ultralytics and `transformers`, and an `/analyze` endpoint returning structured JSON. To keep M0 unblocked, COCO-pretrained classes were mapped to damage categories via a translation dict (a "car" detection became a placeholder for a damaged region). This isn't precise — but it kept the deploy pipeline, frontend integration, and contract tests in motion while a real fine-tune was deferred to M5. [ADR 0001](decisions/0001-pretrained-first.md) documents the trade-off explicitly.

## M2 — Detector chosen and integrated

RT-DETRv2 was picked over YOLOv8 for three reasons: no NMS (it handles overlapping damages better — scratches over dents, broken glass next to bent frames), better sample efficiency on small datasets like CarDD's ~4k images, and a unified Ultralytics API that kept YOLOv8 a one-line fallback. [ADR 0008](decisions/0008-rtdetr-vs-yolo.md). Cold-start was tuned for CPU-only inference — model weights memory-mapped at boot, no per-request load overhead. [ADR 0009](decisions/0009-cold-start-cpu.md).

## M3 — Segmenter and severity logic

SAM 2 was added as a zero-shot segmenter prompted by detection bboxes — no fine-tune needed because the mask quality from SAM 2's general training already exceeds what's gainable from CarDD's limited mask annotations. [ADR 0010](decisions/0010-sam2-zero-shot.md). Severity became an explicit, multiplicative heuristic — `score = area_ratio × type_weight × position_weight` — instead of a learned model, because CarDD has no severity ground truth and the heuristic is auditable to a senior reviewer. [ADR 0011](decisions/0011-severity-heuristic.md). Mask encoding was capped to ≤256-point polygons to fit a 100 KB JSON budget per analyze response. [ADR 0012](decisions/0012-mask-encoding-polygon-cap.md).

## M4 — Frontend polish

Glassmorphism dark theme with hand-curated design tokens (no `fonts.googleapis.com` calls — fonts shipped as woff2 to keep the demo offline-capable) [ADR 0013](decisions/0013-tokens-fonts-strategy.md), lazy-loaded ECharts to keep the initial bundle under 150 KB gzip [ADR 0014](decisions/0014-charts-lazy-load.md), motion budget with discrete (not decorative) Framer Motion animations [ADR 0015](decisions/0015-motion-budget.md). The backend got a strict image budget: 10 MB max, JPEG/PNG/WEBP validated by magic bytes, downscale before inference. [ADR 0016](decisions/0016-backend-image-budget-cpu-torch.md).

## M5 — Fine-tune on CarDD

RT-DETRv2-l was fine-tuned on the CarDD train split (2,816 imgs, 6,211 labels) using a Colab T4: 50-epoch budget, early-stop at epoch 40, AdamW with `lr0=1e-4`, batch 8, `imgsz=640`. Pure `model.train()` time totalled **74 min** across two sessions — Colab Free disconnected mid-training and the second session resumed cleanly via `resume=True` from the persisted `last.pt`. Final verdict on the test split: **mAP@0.5 = 0.701**, mAP@0.5:0.95 = 0.536, CPU latency 614 ms mean (within 1.2× of baseline). Operational deviations (Drive FUSE rejecting symlinks, paths containing spaces, Colab disconnect mid-training) each motivated their own commit — none required hyperparameter changes. [ADR 0017](decisions/0017-finetune-cardd-result.md) · [`metrics/baseline-vs-finetuned.md`](metrics/baseline-vs-finetuned.md).

## M6 — Public delivery

End-to-end deploy to `cca.imb-central.tech` on a self-hosted Coolify instance. nginx sat in front of the FastAPI container [ADR 0006](decisions/0006-frontend-serving-nginx.md) · [ADR 0007](decisions/0007-prod-connectivity-nginx-proxy.md). A Marp-based slide deck was built for interview presentation [ADR 0018](decisions/0018-deck-tooling-marp.md). The demo is publicly accessible; no auth, rate-limited per IP.

## M7 — Image collection and continued training

The biggest scope expansion. The original architecture flagged "no persistence" as a stated out-of-scope item; M7 flipped that — but only for images with explicit consent — to enable continued training on user-donated examples. [ADR 0019](decisions/0019-flip-no-persistence.md). This unlocked four supporting decisions:

- Full LFPDPPP (Mexican data-protection law) compliance — consent capture, ARCO rights endpoint, EXIF stripping, opt-out flow [ADR 0020](decisions/0020-lfpdppp-compliance-architecture.md).
- SQLite + Docker volume for storage (no Postgres / S3 overkill at portfolio scale) [ADR 0021](decisions/0021-storage-sqlite-volume.md).
- Resend for transactional retraining-completion emails [ADR 0022](decisions/0022-email-resend-transactional.md).
- A semi-automatic retraining loop blending donated images back into the CarDD training set [ADR 0023](decisions/0023-retraining-loop-semi-auto.md).

A late M7.1 amendment moved consent from opt-in to default-on with explicit opt-out, after weighing engagement signal against legal posture. [ADR 0024](decisions/0024-consent-default-on-with-opt-out.md).

## M8 — Frontend redesign

The landing experience was refactored to support portfolio positioning. Motion was rebalanced from page-flow scroll-heavy patterns to a more discrete budget [ADR 0025](decisions/0025-motion-budget-flow-vs-scroll.md). Stack and ADR sections were added so the demo itself surfaces the engineering depth without forcing a visitor to leave the page.

---

For numerical detail, see [`metrics/baseline-vs-finetuned.md`](metrics/baseline-vs-finetuned.md) and the raw JSON. For architectural detail, see [`architecture.md`](architecture.md).
