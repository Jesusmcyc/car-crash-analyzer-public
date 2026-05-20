# ADR 0021 — Storage: single-file SQLite + Docker volume

**Status:** Accepted
**Date:** 2026-05-18
**Author:** Jesús Moreno
**Milestone:** M7

## Context

[ADR 0019](0019-flip-no-persistence.md) decides to persist the subset of consented images + their metadata. [ADR 0020](0020-lfpdppp-compliance-architecture.md) imposes the minimization and privacy-by-design requirements. The technical detail remains to be settled: **where and how those bytes live**.

Three realistic options for portfolio scale (expected <10k contributions in the first 6 months, ~5 GB of JPEG q=85 images):

1. **SQLite + Docker volume** on Coolify (the same droplet as the backend).
2. **Postgres** (Coolify-orchestrated) + **S3-compatible** storage (DigitalOcean Spaces $5/month) for the images.
3. **Postgres bytea** for images inline in the DB.

## Decision

**Option 1: single-file SQLite + Docker volume.**

- **Images:** Coolify volume `contributions-images` mounted at the backend's `/app/contributions/`. Each approved image lives at `/app/contributions/images/{sha256}.jpg`.
- **Metadata DB:** SQLite at `/app/contributions/db.sqlite` within the same volume. WAL mode (`PRAGMA journal_mode=WAL`). `synchronous=NORMAL` (the default is FULL; NORMAL is safe with WAL and reduces fsync hits).
- **Schema bootstrap at the backend's lifespan startup** — idempotent (`CREATE TABLE IF NOT EXISTS`).
- **Operational backup:** a monthly manual scp by the operator (`db.sqlite` + the `images/` folder) to their own filesystem. The same procedure as the upload of `rtdetr-cardd.pt` in M5 T9 — the operator already has an SSH key configured (M5 T9 upload section).

**Initial schema (5 tables, see the migration in `backend/app/infra/db.py`):**

- `schema_version` — the schema version, for future manual migrations.
- `subscribers` — registered emails with an `opaque_id` UUID4 for unsubscribe URLs.
- `contributions` — one row per unique image (UNIQUE sha256), optional FK to a subscriber, status enum.
- `training_runs` — one row per trained model (unique `model_version`).
- `training_contributions` — many-to-many between contributions and training_runs.
- `rate_limit_buckets` — `(ip_hash, bucket_day) → count` for the 10/day contribution rate limit.

## Consequences

### Positives

- **Zero additional cost.** A Docker volume is free (limited by the droplet's disk). DigitalOcean Spaces is $5/month recurring + transfer.
- **Operational simplicity.** A single file (`db.sqlite`) openable with the `sqlite3` CLI, DBeaver, or DataGrip. Transfer via scp. No additional daemon to monitor.
- **Same pattern as M5 T9** (the `finetuned-models` volume). The operator already has the playbook documented in `models/MANIFEST.md`.
- **WAL mode supports concurrent reads without a lock.** For portfolio scale (~tens of writes/day, hundreds of reads/day), SQLite WAL is performance overkill.
- **Transactional atomicity for withdraw.** `withdraw_subscriber` updates rows and returns the set to delete on disk — all atomic within the transaction.
- **Schema in code (not in a migration tool).** For portfolio scale, `CREATE TABLE IF NOT EXISTS` is sufficient. Alembic is deferred until a real schema change requires it.
- **Sha256 as natural deduplication.** Same visitor uploads the same image twice → the `UNIQUE(sha256)` constraint deduplicates automatically; the second call updates `uploaded_at` and does not create a row.

### Negatives

- **Scale limited to the droplet's disk.** If contributions exceed 50 GB, a migration to S3 is needed. Mitigation: a monthly `df -h` check; the current estimate of ~5 GB / 10k contributions leaves a 10× margin over a typical disk.
- **Automated backup not implemented.** Month by month, the operator runs a manual scp. If more than 1 month passes without a backup and the droplet fails, there is data loss. Mitigation: documented as a maintenance task.
- **Concurrent writes are serialized.** WAL allows concurrent reads, but writes are globally serialized. For portfolio scale this is a non-issue; for 100+ writes/second it would not apply.
- **No geographic replication.** If the entire droplet is lost, the data is lost too. Acceptable for a portfolio (the trained model lives on the operator's filesystem after each batch; the dataset can be regenerated).

### Neutral

- **SQLite on a Docker volume works in Windows dev** via Docker Desktop bind mounts. Local tests use `tmp_path` in pytest, not the real path.
- **Images live on the filesystem, not in bytea.** An industry standard. It allows direct access from scripts (weekly export, offline annotation) without going through SQL.
- **The volume survives a container restart** (Docker named volume) but does NOT survive a `docker volume rm`. The operator does not run `docker volume rm` except intentionally.

## Alternatives considered

### Option 2: Postgres + S3-compatible

- **Postgres on Coolify:** another service to orchestrate, an additional healthcheck, an additional backup. For a single operator this is unjustified overhead.
- **DigitalOcean Spaces ($5/month):** a recurring cost + the boto3 SDK + credential management + additional latency (vs the local filesystem). The gain: geo-replicated durability — overkill for portfolio scale.
- **Reconsideration:** if the volume exceeds 50 GB or if multi-region is needed (does not apply to a portfolio).

### Option 3: Postgres bytea for images inline

- **A recognized anti-pattern.** Images belong on the filesystem, not in the DB. Bytea inflates backups and queries and degrades IO. The Postgres docs explicitly recommend filesystem + reference for blobs >1 MB.
- **Rejected in discovery, not in design.**

### Sub-alternatives evaluated and discarded

- **DuckDB** instead of SQLite. DuckDB excels at analytics (columnar OLAP); for typical transactional CRUD, SQLite remains the default. There is no advantage for this case.
- **Encryption at rest** for the volume (LUKS, eCryptfs). Defense via access control to the droplet is sufficient for portfolio scale; adding crypto shifts the problem to key management without reducing the risk proportionally.
- **`PRAGMA synchronous=FULL`** (the default). Considered for maximum durability; rejected because it adds an fsync per write and the maximum loss with NORMAL is ~milliseconds (acceptable for a portfolio).

## References

- design doc — D2 (storage decision).
- [ADR 0019](0019-flip-no-persistence.md) — the decision to persist.
- [ADR 0020](0020-lfpdppp-compliance-architecture.md) — privacy by design in the schema.
- **SQLite WAL:** <https://www.sqlite.org/wal.html>
- **SQLite when to use:** <https://www.sqlite.org/whentouse.html>
- **PostgreSQL bytea anti-pattern:** <https://wiki.postgresql.org/wiki/BinaryFilesInDB>
