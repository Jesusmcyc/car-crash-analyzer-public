# ADR 0022 — Transactional email: Resend + a separate subscribers table

**Status:** Accepted
**Date:** 2026-05-18
**Author:** Jesús Moreno
**Milestone:** M7

## Context

M7 spec D7 requires sending an automatic email to the donor when their image contributes to a new version of the model (operator decision 7a, see the `project-m7-decisions` memory). Derived requirements:

- **Expected volume:** 3-12 emails/year (the expected cadence of model v2/v3/v4) × N donors with a registered email. Realistically: <500 emails/year.
- **Transactional, not marketing:** one email per model release, not a newsletter.
- **Reasonable deliverability:** donors should receive the email in their inbox, not in spam.
- **Trivial unsubscribe:** a legal requirement under LFPDPPP (the right of cancellation) + CAN-SPAM (USA, where Resend is based).
- **Low cost:** a free tier is sufficient; do not burden the demo with recurring costs.

Four providers were evaluated: **Resend**, Postmark, SendGrid, Mailgun. Listmonk (self-hosted) and direct SMTP via Gmail were also considered.

## Decision

**Resend** as the transactional provider + a **separate `subscribers` table** from `contributions` in SQLite.

### Provider: Resend

- **Free tier:** 3,000 emails/month, 100/day — more than enough for the expected volume.
- **Simple API:** `resend.Emails.send({from, to, subject, text, html})`. Official Python SDK (`pip install resend`).
- **Initial sender:** `onboarding@resend.dev` (the Resend default, requires no DNS). Documented plan B: `@cca.imb-central.tech` with DNS verification (SPF + DKIM TXT records, 24-48h propagation).
- **API key in the `RESEND_API_KEY` env var.** If it is not set, there is an automatic fallback to `LocalLogEmailSender`, which logs but does not send (useful for local dev + tests).

### Schema: a separate `subscribers` table

```sql
CREATE TABLE subscribers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    email TEXT NOT NULL UNIQUE,
    opaque_id TEXT NOT NULL UNIQUE,    -- UUID4 para URLs de unsubscribe
    subscribed_at TEXT NOT NULL,
    unsubscribed_at TEXT,
    unsubscribe_reason TEXT
);

ALTER TABLE contributions ADD COLUMN subscriber_id INTEGER REFERENCES subscribers(id);
```

**Email in plain text in a private DB.** Not hashed, not encrypted at rest. The real defense is access control to the Coolify volume; additional encryption shifts the problem without reducing the risk proportionally for portfolio scale.

**Disclosure in the privacy notice** (the literal text to be published at `/privacidad`):

> Tu email se almacena en plain text en un volumen privado del operador. Lo usamos solo para notificarte cuando una nueva versión del modelo entrene con tu imagen. No te suscribimos a newsletter ni compartimos con terceros. Puedes retirar el consentimiento en cualquier momento.

### Sending pipeline

Single-shot and manual (`scripts/notify_contributors.py`) — the operator runs it after deploying v2/v3/v4 with the `model_version` as an argument:

```bash
python scripts/notify_contributors.py --model-version=rtdetr-cardd-v2
python scripts/notify_contributors.py --model-version=rtdetr-cardd-v2 --dry-run  # preview sin enviar
```

Internal query: `SELECT email, opaque_id FROM subscribers JOIN contributions ON ... JOIN training_contributions ON ... WHERE training_runs.model_version = ? AND subscribers.unsubscribed_at IS NULL` — automatic dedup by email.

## Consequences

### Positives

- **The free tier is more than enough.** 3,000/month vs the expected <500/year = a 60× margin. The free tier is permanent as long as it does not scale; the $20/month plan if it ever becomes necessary.
- **Modern DX.** Resend has a better Python SDK than SendGrid v1 (deprecated) or v2 (clunky). Postmark is competitive but requires a more extensive setup.
- **A separate table enables minimization.** If a contribution has no email, it does not enter `subscribers`. Multiple contributions from the same email link to a single `subscriber.id`, with no duplication.
- **Opaque ID = robust UUIDs for unsubscribe.** They do not expose the email or the internal ID in URLs or logs.
- **Automatic fallback to `LocalLogEmailSender`** without `RESEND_API_KEY`. Tests require neither elaborate mocks nor internet.
- **Dry-run mode** in the script: the operator can see "who it would send to" before sending.

### Negatives

- **Dependency on a third party (Resend Inc., USA).** If Resend deactivates the account or changes pricing, a migration is needed. Mitigation: the code abstracts an `EmailSender` Protocol; a swap to Postmark/SendGrid is ~30 lines of code.
- **International data transfer.** The email is transferred to the USA (Resend's servers). Declared in the LFPDPPP privacy notice — it is not a non-issue, but the legal basis is the data subject's express consent in the upload flow.
- **The generic default sender** (`onboarding@resend.dev`) may trigger antispam filters in some clients. An upgrade to the custom sender `@cca.imb-central.tech` remains a post-M7 operational task if deliverability becomes an issue.
- **The plain-text emails are human-readable** — accessible to the operator with SSH to the droplet. By design (necessary for the send), not by accident.

### Neutral

- **The operator's Resend account** requires a manual, one-time setup: signup + API key + paste into the Coolify UI. Documented in plan T15.
- **Tests do not use a real network.** `LocalLogEmailSender` covers the success path + a conceptual failure path. A smoke test in prod with the real API key is the actual integration test.
- **The email is sent only when the operator triggers the script.** No cron, no auto-send after training. It keeps human control over the timing.

## Alternatives considered

### Postmark

- More expensive ($15/month free → $15/month for 10k emails) but with excellent deliverability + a more mature setup.
- Rejected for portfolio scale — the Resend free tier is enough.
- **To be reconsidered as plan B** if Resend fails.

### SendGrid

- A free tier of 100/day (similar to the Resend free tier).
- The Python SDK has a history of brittle versioning (v1 deprecated, v2 clunky).
- Rejected for inferior DX compared to Resend at the same price.

### Mailgun

- Switched to a 30-day paid trial in 2023; there is no persistent free tier.
- Rejected on cost.

### Listmonk self-hosted

- An open-source mailing-list manager.
- Rejected: it adds a Coolify service, requires its own SMTP, and is overkill for 3-12 sends per year.
- **To be reconsidered** if the volume scales to multiple segmented newsletters (which is not the case).

### Direct SMTP via Gmail (App Password)

- Free, simple.
- Rejected: Gmail rate-limits outbound SMTP, it is not intended for transactional use at scale, and deliverability is unpredictable (DKIM/SPF dependent on personal sending). A professional no-go.

### Email in plain text vs encrypted at rest

- **Plain text** (the decision): minimal maintenance, the send requires the plaintext, and additional encryption shifts the problem to key management.
- **`cryptography.Fernet` with a key in an env var**: adds key rotation + recovery if the operator loses the key. Operational cost > benefit for portfolio scale.
- **Email hash** (`bcrypt`/`argon2`): breaks proactive sending — the email cannot be recovered from the hash. It would only serve for deduplication, which we already cover with `UNIQUE(email)`.

## References

- design doc — D3 (email handling), D7 (notification mechanism).
- [ADR 0020](0020-lfpdppp-compliance-architecture.md) — declaration of the international transfer to the USA.
- [ADR 0021](0021-storage-sqlite-volume.md) — the `subscribers` table in the SQLite schema.
- **Resend Python SDK:** <https://resend.com/docs/sdks/python>
- **Postmark (plan B):** <https://postmarkapp.com>
- **CAN-SPAM (USA):** <https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business>
