# ADR 0020 — LFPDPPP compliance architecture

**Status:** Accepted
**Date:** 2026-05-18
**Author:** Jesús Moreno
**Milestone:** M7

## Context

[ADR 0019](0019-flip-no-persistence.md) lifts the "no persistence" constraint for the subset of consented images in M7. This opens up legal exposure under the Mexican personal-data protection regime:

- **Ley Federal de Protección de Datos Personales en Posesión de los Particulares (LFPDPPP)** (Federal Law on the Protection of Personal Data Held by Private Parties), in force since 2010, is the applicable jurisdiction because (a) the data controller is a Mexican natural person (Jesús Moreno), (b) the data subjects may be natural persons within Mexican territory, (c) the personal data collected includes email (when provided) and images that may contain vehicle license plates, faces, or EXIF location (which can be categorized as sensitive data).
- **Supervisory authority:** INAI (Instituto Nacional de Transparencia, Acceso a la Información y Protección de Datos Personales).
- **Potential sanctions:** administrative fines + individual claims for damages.

GDPR does not strictly apply (there is no establishment in the EU), but the practices are compatible. LFPDPPP is stricter on some points (express consent for sensitive data).

## Decision

Three compliance pillars, all implemented as part of M7:

### Pillar 1 — Accessible, specific privacy notice

- **`/privacidad` page** (SPA route, not a PDF) with the 8 sections required by LFPDPPP art. 16:
  1. Identity and address of the data controller.
  2. Personal data collected.
  3. Sensitive personal data (explicit declaration: images may contain license plates/faces/EXIF GPS).
  4. Purposes (a single one: model re-training).
  5. Transfers (email to Resend Inc. in the USA, declared).
  6. ARCO rights + procedure to exercise them.
  7. Means to limit use (not checking consent + withdraw endpoint).
  8. Changes to the notice (versioning + re-consent if a substantive change occurs).
- **Spanish text**, written by the operator (not an auto-generated template).
- **Versioning:** every contribution stores `consent_version` (string `"privacy-v1"`); substantive changes bump the version and trigger re-consent.
- **Linked from three places:** the consent checkbox, the global footer, and every transactional email (D7 spec M7).

### Pillar 2 — Privacy by design in processing

- **EXIF strip before persisting.** `ImageOps.exif_transpose` to honor visual orientation, then a new image without EXIF (D5 spec M7). The sha256 is computed over the post-strip bytes — the persisted file never contains GPS data or a device serial.
- **Resize to 1024 px** (longer side, Lanczos) and **recompress as JPEG q=85**, unified — minimization of the persisted data while keeping it useful for training.
- **No facial recognition, no license-plate OCR.** The CV pipeline detects damage types (`dent`, `scratch`, etc.) — it does not identify individual people or vehicles.
- **IP hash with a daily UTC salt** — `sha256(ip + "|" + YYYY-MM-DD)`. Allows intra-day forensics for rate limiting; prevents longitudinal tracking.
- **Email in a separate `subscribers` table** with an `opaque_id` UUID4 for unsubscribe URLs. No exposure of email in logs or headers.

### Pillar 3 — Operational ARCO rights

- **`A`ccess + `R`ectification:** via email to `jesusmcyc@gmail.com` with the contribution's `request_id` (captured in the post-upload toast). Processed manually within ≤20 business days (LFPDPPP).
- **`C`ancellation:** public, idempotent endpoint `POST /api/privacy/withdraw/{opaque_id}`. For contributors with a registered email, the opaque_id is included in every transactional email. For anonymous contributors (no email), cancellation is handled via a manual email to the operator.
- **`O`bjection:** equivalent to not checking the consent checkbox. The visitor can use the demo without contributing.
- **Hard delete of the physical file** in the withdraw endpoint (`os.unlink`). LFPDPPP requires effective cancellation — a soft delete with a flag does not qualify if the data remains accessible.
- **Idempotency:** a second call to the same opaque_id returns `{status: "already_withdrawn"}`. No errors on a double click.

## Consequences

### Positives

- **Defense against a realistic INAI claim.** A complete notice + operational ARCO + demonstrable minimization = a solid position should a claim arrive.
- **Visible trust for LinkedIn visitors.** A Spanish-language notice accessible from the footer communicates operational seriousness.
- **EXIF strip also protects the operator.** Even though the visitor uploads voluntarily, the operator does not want to store GPS coordinates or device serials.
- **A separate `subscribers` table simplifies auditing.** Seeing "all contributions from this email" is a SELECT JOIN; minimizing is a DROP ROW.

### Negatives

- **Maintaining the notice on every substantive change.** If the purpose expands (e.g. "share the dataset with a third party"), the version bumps + re-consent. The operational cost is not zero.
- **The public withdraw endpoint is attackable.** Mitigation: the opaque_id is a UUID4 (impractical to guess). Residual risk is acceptable.
- **EXIF strip removes information that could be useful for future training** (e.g. camera, lens, or date metadata). An explicit trade-off in favor of privacy.

### Neutral

- **The privacy notice is the operator's human responsibility.** Sensitive legal text requires a human signature; the initial draft goes through review before the `<!-- DRAFT -->` marker is removed.
- **There is no representative in Mexican territory** — not required for a natural person resident in Mexico.
- **There is no formal registration with INAI** — LFPDPPP does not require prior registration (unlike some European regimes).

## Alternatives considered

1. **Auto-generated legal template (Iubenda, TermsFeed).** Rejected: $5-15/month recurring, templates are often broader than the actual use (they declare data sharing with third parties that never happens), and they generate distrust in careful readers.
2. **Privacy notice as an embedded PDF.** Rejected: not versionable with a line-by-line diff, requires opening in an external viewer, less accessible on mobile.
3. **Soft delete with a purge job after 30 days** instead of an immediate hard delete. Rejected — LFPDPPP "cancellation" does not allow an arbitrary delay without a legitimate justification.
4. **Skipping the right of objection** (assuming that "not checking the checkbox" is enough). Rejected for completeness — declaring the means explicitly provides legal protection.
5. **Auto-blur of faces and license plates with CV pre-persist.** Deferred to future iterations — express consent + EXIF strip + privacy notice cover the case. Auto-blur adds complexity without reducing legal risk proportionally.

## References

- design doc — D4 (notice), D5 (privacy by design), D9 (withdrawal).
- [ADR 0019](0019-flip-no-persistence.md) — the "no persistence" flip that opens this exposure.
- [ADR 0022](0022-email-resend-transactional.md) — transfer of the email to Resend USA (declared in the notice).
- **LFPDPPP official text:** <https://www.diputados.gob.mx/LeyesBiblio/pdf/LFPDPPP.pdf>
- **INAI:** <https://home.inai.org.mx/>
- **LFPDPPP regulations:** <https://www.diputados.gob.mx/LeyesBiblio/regley/Reg_LFPDPPP.pdf>
