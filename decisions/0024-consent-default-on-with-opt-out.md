# ADR 0024 — Consent flow: default-on with visible opt-out

**Status:** Accepted
**Date:** 2026-05-18
**Author:** Jesús Moreno
**Milestone:** M7.1 (UI polish post-M7)

## Context

[ADR 0019](0019-flip-no-persistence.md) made persistence conditional on express `consent`. [ADR 0020](0020-lfpdppp-compliance-architecture.md) implemented the initial M7 flow: an **opt-in default-off** checkbox ("I agree to have my image stored…"). M7 spec D1 cemented that decision as "legally safer" at the time.

After deploy + prod smoke test, the operator identified three issues with the opt-in default-off model:

1. **Friction for the target visitor (LinkedIn/portfolio).** The page delivers value with the analysis; a checkbox separate from the main flow adds an extra cognitive step.
2. **Low expected donation rate (~5-15% per UX benchmarks).** The dataset grows slowly → little data for fine-tuning in subsequent cycles.
3. **The checkbox copy** ("I agree to have my image stored to improve the model") read as an explicit commitment; many visitors skip it out of caution even though the demo is low-risk.

The operator proposed switching to a model where "by uploading the image the user agrees to donate it" (implied consent through use). After review:

- **Pure implied consent** (no banner) does not satisfy LFPDPPP art. 8 for data that may be sensitive (images with license plates, faces, EXIF GPS).
- **Express consent through affirmative action** (banner + click of the upload button) does satisfy it, provided the banner is unambiguous and the visitor has a visible option to object (LFPDPPP arts. 8 and 11).

## Decision

Migrate to the **default-on with visible opt-out** model:

1. **A prominent banner** above the dropzone declaring that uploading the image will analyze **and store** it to improve the model. Inline link to the privacy notice.
2. **An optional email field** available (enabled) by default; useful for post-training notification.
3. **A subtle opt-out checkbox** at the end of the block: *"Do not store my image — analyze only"*. If the visitor checks it before uploading, the backend receives `consent=false` and discards the image after the analysis (zero persistence).
4. **The act of uploading** = express consent through affirmative action (assuming the opt-out is unchecked).

The backend does not change: the `POST /api/analyze` contract already accepts `consent: bool = False` as a query field. Only the frontend changes the default to `consent=true` and the UI component to banner + opt-out.

## Consequences

### Positives

- **Expected donation rate ~80-95%.** A visitor who only wants to try the demo (not donate) can do so with one extra click; the rest contribute through the natural flow.
- **The dataset grows faster** → more frequent fine-tuning cycles → faster model improvement (visible in LinkedIn posts about "v2 trained with N donated images").
- **Cleaner UX** — a single banner instead of a checkbox + legalese label.
- **The email field is enabled by default** without harming UX: the visitor simply does not fill it in if they do not want to be notified.

### Negatives

- **An LFPDPPP trade-off compared to the original M7.** We move from "express consent via an active checkbox" to "express consent via affirmative action with a banner". The latter is accepted INAI case law (clicks as affirmative action for non-sensitive data + a banner declaring the use) **provided the banner is unambiguous and the opt-out is visible**. Both requirements are met. **If an INAI reviewer questioned the change, the updated privacy notice sec 7 explains the model in clear language.**
- **Visitors who do not read the banner** could contribute without noticing — the "burden of reading" shifts to the visitor. Mitigation: the banner is large, requires no scrolling, and uses a clear verb ("is stored"). The withdraw endpoint remains available for immediate removal.
- **The privacy notice sec 7 changes** — versioning via `CONSENT_VERSION` is not incremented because the notice describes the same processing; only the means of expressing consent changes. Documented here + in the commit.

### Neutral

- **API backward compatibility:** `POST /api/analyze` still accepts `consent: bool = False` as optional. The old frontend (M7 with the opt-in checkbox) would still work against the new backend with no changes.
- **The withdraw endpoint** (`POST /api/privacy/withdraw/{opaque_id}`) remains the primary mechanism for ARCO cancellation.
- **EXIF strip + resize + recompress** (ADR 0020) still applies — privacy by design preserved.

## Alternatives considered

1. **Status quo (original M7) — opt-in default-off with warmer copy.** Rejected: the operator validated that the cognitive friction of the separate checkbox is the real blocker, not just the copy.
2. **Pure implied consent (banner without opt-out).** Rejected: LFPDPPP art. 11 requires a simple means of objection for all consent models, and spec D9 already documented that pre-checked / no opt-out is a direct violation.
3. **A dual upload button ("Analyze only" vs "Analyze and donate").** Rejected because it introduces decision paralysis and increases UI complexity for a decision that few people make consciously on first use.

## Risks / mitigations

- **Risk:** A visitor files a complaint with INAI arguing that they did not realize their image was being stored. **Mitigation:** a clear Spanish banner + an always-visible link to the notice + a visible opt-out in the same viewport + an operational withdraw endpoint + a documented ARCO procedure via email. A solid defensive position.
- **Risk:** An overly high donation rate fills up the disk. **Mitigation:** monthly monitoring of the Coolify volume (already documented in ADR 0021); a rate limit of 10 contributions/IP/day (already in place).
- **Risk:** The change confuses users who already knew the original M7 flow. **Mitigation:** the updated privacy notice sec 7 declares the active model; recurring visitors (unlikely in a portfolio) see the difference immediately.

## References

- [ADR 0019](0019-flip-no-persistence.md) — the original "no persistence" flip.
- [ADR 0020](0020-lfpdppp-compliance-architecture.md) — the base LFPDPPP architecture.
- design doc sec D1 — the original opt-in model superseded by this ADR.
- `frontend/src/components/upload/ConsentBlock.tsx` — implementation.
- `frontend/src/pages/PrivacyPage.tsx` sec 7 — updated privacy notice.
- **LFPDPPP official text:** <https://www.diputados.gob.mx/LeyesBiblio/pdf/LFPDPPP.pdf>
