# ADR 0023 — Retraining loop: weekly export cron + manual annotation + retrain in Colab

**Status:** Accepted
**Date:** 2026-05-18
**Author:** Jesús Moreno
**Milestone:** M7

## Context

M7 spec D6 defines the cadence for retraining the model with donated images. The operator is a single person (Jesús); there is no ML ops team. The compute budget is Colab Pay As You Go ($10 unblocked M5 — a proven pattern).

Three options were evaluated for the cadence (5a/5b/5c from the original menu):

- **5a — Manual batched.** The operator downloads monthly, annotates offline, and retrains in Colab.
- **5b — Semi-auto with cron** (operator's decision). A weekly cron exports new rows to an annotation folder; the operator runs the annotation + retrain whenever they decide.
- **5c — Auto-label with confidence threshold.** The current model auto-annotates anything it detects with high confidence; low-confidence detections go to a manual queue.

## Decision

**Option 5b — Semi-auto with weekly export cron + manual annotation + retrain in Colab.**

### Operational flow

1. **Self-scheduled cron** in a separate `cron-export` container runs every Sunday at 02:00 UTC.
2. **Export** of the rows with `status = 'pending'` to `/app/contributions/exports/YYYY-WNN/`:
   ```
   exports/2026-W22/
   ├── manifest.csv     # id, sha256, uploaded_at, status, request_id
   ├── images/          # symlinks to /app/contributions/images/{sha256}.jpg
   └── README.md        # how to download it + annotation + retrain (operator-facing)
   ```
3. **The operator downloads** via manual scp when they decide to process the batch (weekly, monthly — their choice).
4. **Offline annotation** in Roboflow (free tier ≤1000 imgs/project) or LabelImg locally. Output in COCO format.
5. **COCO → YOLO** via the existing script `scripts/cardd_coco_to_yolo.py` (extended if necessary).
6. **Merge into the CarDD training set** + new batch, retrain in Colab (M5).
7. **Eval against the original CarDD test split** (do not include new contributions in the eval — avoids leak).
8. **Deploy verdict** if the D7 thresholds from the M5 spec are met.
9. **Update the weights** (`rtdetr-cardd.pt`) on the Coolify volume via scp (same pattern as M5 T9).
10. **Mark `contributions.status = 'trained'`** for the rows used + insert into `training_runs` and `training_contributions`.
11. **Notify subscribers** with `python scripts/notify_contributors.py --model-version=rtdetr-cardd-v2`.

### Technical implementation

- **Self-scheduled cron** (no crond, no ofelia): the `export_weekly_batch.py` script runs a loop `while True: sleep_until_next_sunday_2am_utc(); run_once()`. A container restart loses the sleep, but the next Sunday it runs normally. Simple, with no dependencies.
- **Symlinks in the export folder** (not copies) to avoid duplicating bytes.
- **`manifest.csv` with sha256** so the operator can verify integrity after scp.
- **A README.md per batch** with operational instructions: how to annotate, which class corresponds to what, what to do if the image is garbage.

## Consequences

### Positives

- **Manual annotation preserves dataset quality.** Auto-label with a confidence threshold introduces bias (the model learns to confirm what it already believes); semi-auto with propagation + revision is the industry standard.
- **A human verdict gate before deploying** the new weights. M5 established pre-committed thresholds (D7 of the M5 spec); this loop inherits those thresholds. The operator does not deploy garbage just because.
- **The operator controls the timing of the retrain.** The cron only exports; it does not consume Colab compute units on its own. The operator decides when to spend (typically when the accumulated batch justifies the cost).
- **The same workflow proven in M5.** The `T7-runbook.md` notebook already works; this loop is a repeat of M5 with a merged dataset.
- **The `cron-export` container is separate from the backend.** If the cron has a bug and crashes, the backend keeps running. If the backend crashes, the cron keeps exporting (data stays `pending`).

### Negatives

- **The human cadence can become saturated.** If the batch grows faster than the operator's attention (e.g. the demo goes viral on LinkedIn), `pending` rows pile up. Mitigation: a manual monthly monitor with `SELECT COUNT(*) WHERE status='pending'`.
- **There is no active learning** (the model does not choose which samples it needs most). The operator annotates whatever arrived, not what is most useful. An explicit trade-off.
- **Human annotation is slow** (~30s/img is reasonable). 100 imgs = 50 min of work. Mitigation: the operator annotates the batch that contributes the most (skipping ambiguous or garbage images).
- **If the `cron-export` container dies during the export**, that specific batch is left incomplete. Mitigation: the script is idempotent — the next run reprocesses the pending rows.

### Neutral

- **The Roboflow free tier** is sufficient for portfolio scale (≤1000 imgs/project). Plan B: LabelImg locally if Roboflow privacy is an issue.
- **The weekly batch may be empty** (no new contributions). The script generates an empty CSV and does not fail — exporting nothing is valid.
- **`status='trained'` enables analytics** ("how many contributions have contributed to a real model"). Useful for LinkedIn posts and for the operator's internal metrics.

## Alternatives considered

### 5a — Manual batched (no cron, operator downloads directly)

- Simpler: the operator runs `sqlite3 db.sqlite ".dump"` + `scp images/` whenever they decide.
- Rejected because of the "what exactly to download" cost — the cron pre-packages the batch, manifest, and README. It reduces operational friction.

### 5c — Auto-label with confidence threshold

- The current model processes every contribution; if `confidence > 0.8` for a class, it auto-annotates; otherwise it goes to a manual queue.
- Rejected: confirmation bias (the model reinforces its own biases). The per-class results from M5 show that `scratch` is at mAP 0.51 — auto-annotating what the model believes are scratches will reinforce its systematic error.
- **Reconsiderable** if a "model drift" metric is established and the operator has the budget to review the manual queue at the required cadence.

### Daily cron instead of weekly

- Unnecessarily granular. The operator does not annotate daily.
- Rejected.

### Auto-triggering the retrain when the batch exceeds N

- Attractive in the abstract but dangerous: the retrain consumes Colab Pay As You Go = real $ cost. The operator wants human control over when to spend.
- Rejected for M7. Reconsiderable for M8+ if the cycle becomes predictable.

### Annotation with custom tools (e.g. self-hosted Label Studio)

- More control + more overhead. Roboflow free + LabelImg locally cover the case.
- Rejected because of operational cost.

### Eval including new contributions in the test split

- Tempting (more data → better eval).
- Rejected: an inevitable leak. The original CarDD test split is the invariant for comparing epochs/runs. New contributions go only into train.

## References

- design doc — D6 (retraining cadence).
- design doc — D7 thresholds verdict (inherited).
- [ADR 0017](0017-finetune-cardd-result.md) — deploy verdict of the first fine-tune (template for the following ones).
- **Roboflow:** <https://roboflow.com>
- **LabelImg:** <https://github.com/HumanSignal/labelImg>
