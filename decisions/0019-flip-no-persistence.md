# ADR 0019 — Flipping "no persistence" to enable continued training

**Status:** Accepted
**Date:** 2026-05-18
**Author:** Jesús Moreno
**Milestone:** M7

## Context

The `architecture doc` sec 11 had listed, since M0, among the out-of-scope items:

> - Persistence (DB). Inferences are fire-and-forget.

That restriction kept the backend stateless throughout M0–M6: the `/api/analyze` endpoint processes the image in memory, returns the JSON report, and releases all resources. Zero persisted files, zero DB. It was the correct decision for the modeling milestones (M0–M5) and the portfolio delivery (M6).

M7 introduces the **continued training** feature on images voluntarily donated by visitors of the public demo. The M7 decisions (spec D2 + D3) require persisting the subset of images with express consent (LFPDPPP), together with metadata (`sha256`, `uploaded_at`, `consent_version`, optional `subscriber_id`) and the donor's email for post-training notification (D7).

The "no persistence" restriction from the original scope must be lifted in a controlled way for this subset, without opening the door to persisting every inference.

## Decision

**The "no persistence" restriction is lifted ONLY for images with an explicit `consent=true`.** The rest of the pipeline stays stateless as it is today:

| Case | Persistence | Rationale |
|---|---|---|
| `/api/analyze` with `consent=false` (or no field) | **No** | Backward compatible with M5; fire-and-forget |
| `/api/analyze` with `consent=true` and a current consent_version | **Yes** | LFPDPPP — express consent |
| Structured inference logs | Not persisted in DB | Loguru stdout, container-level capture |
| Aggregate operational metrics | Not persisted | Out of scope for M7 |

**Persistence stack (detailed in ADR 0021):**

- **SQLite** single-file (`/app/contributions/db.sqlite`), WAL mode.
- **Docker volume** `contributions-images` mounted at `/app/contributions/` (Coolify-orchestrated, the same pattern as `finetuned-models` from M5 T9).
- **No Postgres, no S3, no Redis** — overkill for portfolio scale.

**Original scope updated:** the "Persistence (DB)" bullet is kept struck through in `architecture.md` with an explicit redirect to this ADR. An operator who arrives cold at the repo understands the nuance from the first read.

## Consequences

### Positives

- **Continued training becomes viable.** Without persistence, the model improvement cycle depended 100% on the original CarDD dataset (4k imgs); with persistence, the cycle extends to the demo's community.
- **The public demo gains a differentiating feature for the portfolio.** "An app that learns from whoever uses it" is a differentiating highlight on LinkedIn versus yet another generic CV demo.
- **Narrative engagement for LinkedIn posts.** Each new version of the model provides publishable material: "v2 trained on N photos donated by the community".
- **Demonstrable operational discipline.** Implementing consent, ARCO rights, EXIF stripping, a withdrawal endpoint, and a complete retraining loop is "production-grade" work that a senior technical reviewer recognizes — stronger than adding more slides to the deck.

### Negatives

- **A new legal surface (LFPDPPP).** Detail in [ADR 0020](0020-lfpdppp-compliance-architecture.md). If the privacy notice is insufficient or the ARCO flow fails, the risk is real (a complaint to INAI, the Mexican authority).
- **Manual operational backup.** The Coolify volume survives a container restart, but the operator must download `db.sqlite` + `images/` monthly to their own filesystem. Automated backup is out of scope for M7 (a potential M7.5).
- **Limited droplet disk.** ~5 GB expected for 10k contributions (resize + JPEG q=85 = ~500 KB/img). Manual monthly monitoring with `df -h`.
- **One more failure path:** SQLite WAL can become corrupted if Coolify kills the container during a write. Mitigation: `PRAGMA synchronous=NORMAL` + WAL mode; maximum estimated loss is on the order of milliseconds.

### Neutral

- **The original scope is updated with a redirect to this ADR** — the "Persistence (DB)" bullet in `architecture.md` is struck through with an inline note. A cold operator reads the nuance on the first pass.
- **The rest of the "out of scope" items remain in force** — auth, i18n, mobile-first, PWA, webhooks. M7 lifts exactly one bullet; it does not open a parenthesis for the others.
- **The CV pipeline (`backend/app/pipeline/`) is not touched.** The M7 feature lives in a new adjacent module, `backend/app/contributions/`. A clear separation so the reviewer understands what is modeling and what is data collection.

## Alternatives considered

1. **Postgres + an S3-compatible store (DigitalOcean Spaces, $5/month).** Rejected — overkill for portfolio scale. SQLite WAL + a Docker volume is sufficient for the first 10k contributions estimated. Reconsider if the volume exceeds 50 GB or if concurrent access from multiple writers is needed (does not apply to a single operator).
2. **Defer the feature to M7.5 after the UI polish.** Rejected — it contradicts the portfolio direction (the `project-portfolio-direction` memory); the polish + paper integrate better with the feature already active than by splitting the milestones apart.
3. **Persist ALL inferences (even without consent), anonymized.** Rejected — LFPDPPP requires express consent for the processing; "anonymization" without the data subject's consent does not hold up as a solid legal basis under the Mexican regime when the data may be potentially identifiable (images with license plates / faces).
4. **Lift the "no persistence" restriction without restricting the use case.** Rejected on the principle of minimization — the M7 decision is specific (continued training, explicit opt-in), not generic.

## References

- `architecture doc` sec 11 — the original restriction (modified in M7).
- design doc — M7 spec D1-D3, D9.
- [ADR 0020](0020-lfpdppp-compliance-architecture.md) — the associated legal compliance.
- [ADR 0021](0021-storage-sqlite-volume.md) — the technical detail of SQLite + volume.
- [ADR 0022](0022-email-resend-transactional.md) — email handling, kept separate.
- [ADR 0023](0023-retraining-loop-semi-auto.md) — the retraining pipeline.
- LFPDPPP: <https://www.diputados.gob.mx/LeyesBiblio/pdf/LFPDPPP.pdf>
