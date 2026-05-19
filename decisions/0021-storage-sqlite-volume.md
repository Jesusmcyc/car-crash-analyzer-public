# ADR 0021 — Storage: SQLite single-file + volumen Docker

**Estado:** Aceptado
**Fecha:** 2026-05-18
**Autor:** Jesús Moreno
**Milestone:** M7

## Contexto

[ADR 0019](0019-flip-no-persistence.md) decide persistir el subset de imágenes consented + su metadata. [ADR 0020](0020-lfpdppp-compliance-architecture.md) impone los requisitos de minimización y privacy by design. Falta resolver el detalle técnico: **dónde y cómo viven esos bytes**.

Tres opciones reales para portfolio scale (esperado <10k contribs en los primeros 6 meses, ~5 GB de imágenes JPEG q=85):

1. **SQLite + volumen Docker** en Coolify (mismo droplet que el backend).
2. **Postgres** (Coolify-orquestado) + **S3-compatible** (DigitalOcean Spaces $5/mes) para las imágenes.
3. **Postgres bytea** para imágenes inline en DB.

## Decisión

**Opción 1: SQLite single-file + volumen Docker.**

- **Imágenes:** volumen Coolify `contributions-images` montado en `/app/contributions/` del backend. Cada imagen aprobada vive en `/app/contributions/images/{sha256}.jpg`.
- **Metadata DB:** SQLite en `/app/contributions/db.sqlite` dentro del mismo volumen. WAL mode (`PRAGMA journal_mode=WAL`). `synchronous=NORMAL` (default es FULL, NORMAL es seguro con WAL y reduce fsync hits).
- **Bootstrap del schema al lifespan startup** del backend — idempotente (`CREATE TABLE IF NOT EXISTS`).
- **Backup operacional:** scp manual mensual del operador (`db.sqlite` + folder `images/`) al filesystem propio. Mismo procedimiento que el upload de `rtdetr-cardd.pt` en M5 T9 — el operador ya tiene SSH key configurada (M5 T9 sección upload).

**Schema inicial (5 tablas, ver migration en `backend/app/infra/db.py`):**

- `schema_version` — versión del schema, para futuras migraciones manuales.
- `subscribers` — emails registrados con `opaque_id` UUID4 para unsubscribe URLs.
- `contributions` — fila por imagen única (UNIQUE sha256), FK opcional a subscriber, status enum.
- `training_runs` — fila por modelo entrenado (`model_version` único).
- `training_contributions` — many-to-many entre contributions y training_runs.
- `rate_limit_buckets` — `(ip_hash, bucket_day) → count` para rate limit de contribución 10/día.

## Consecuencias

### Positivas

- **Costo cero adicional.** Volumen Docker es free (limitado por disco del droplet). DigitalOcean Spaces son $5/mes recurrentes + transferencia.
- **Simplicidad operativa.** Un solo archivo (`db.sqlite`) abrible con `sqlite3` CLI, DBeaver, o DataGrip. Transfer vía scp. Sin daemon adicional que monitorear.
- **Mismo patrón que M5 T9** (`finetuned-models` volume). El operador ya tiene el playbook documentado en `models/MANIFEST.md`.
- **WAL mode soporta concurrent reads sin lock.** Para portfolio scale (~tens of writes/día, hundreds of reads/día), SQLite WAL es performance overkill.
- **Atomicidad transaccional para withdraw.** `withdraw_subscriber` actualiza filas y devuelve el set a borrar en disco — todo atómico dentro de la transacción.
- **Schema en código (no en migration tool).** Para portfolio scale, `CREATE TABLE IF NOT EXISTS` es suficiente. Alembic queda postergado hasta que un schema change real lo requiera.
- **Sha256 como deduplicación natural.** Mismo visitante sube misma imagen 2 veces → `UNIQUE(sha256)` constraint dedup automático; segunda llamada actualiza `uploaded_at`, no crea fila.

### Negativas

- **Escala limitada al disco del droplet.** Si la contribuciones superan 50 GB hay que migrar a S3. Mitigación: monitor `df -h` mensual; estimado actual ~5 GB / 10k contribs deja margen 10× sobre disco típico.
- **Backup automatizado no implementado.** Mes a mes, el operador ejecuta scp manual. Si pasa más de 1 mes sin backup y el droplet falla, hay pérdida. Mitigación: documentado como tarea de mantenimiento.
- **Concurrent writes serializadas.** WAL permite concurrent reads pero writes son globales-serializados. Para portfolio scale es non-issue; para 100+ writes/segundo no aplicaría.
- **No replicación geográfica.** Si el droplet entero se pierde, los datos también. Aceptable para portfolio (el modelo entrenado vive en filesystem del operador post-batch; el dataset puede regenerarse).

### Neutrales

- **SQLite en volumen Docker funciona en Windows dev** vía Docker Desktop bind mounts. Tests locales usan `tmp_path` en pytest, no el path real.
- **Las imágenes viven en filesystem, no en bytea.** Estándar de la industria. Permite acceso directo desde scripts (export semanal, anotación offline) sin pasar por SQL.
- **El volumen sobrevive container restart** (Docker named volume) pero NO sobrevive a `docker volume rm`. El operador no ejecuta `docker volume rm` salvo intencionalmente.

## Alternativas consideradas

### Opción 2: Postgres + S3-compatible

- **Postgres en Coolify:** otro servicio a orquestar, healthcheck adicional, backup adicional. Para single-operator es overhead injustificado.
- **DigitalOcean Spaces ($5/mes):** costo recurrente + SDK boto3 + manejo de credentials + latencia adicional (vs filesystem local). Ganancia: durabilidad geo-replicada — sobrekill para portfolio scale.
- **Reconsideración:** si el volumen supera 50 GB o si se necesita multi-region (no aplica a portfolio).

### Opción 3: Postgres bytea para imágenes inline

- **Anti-pattern reconocido.** Las imágenes pertenecen al filesystem, no a la DB. Bytea infla los backups, los queries, y degrada IO. Postgres docs explícitamente recomiendan filesystem + reference para blobs >1 MB.
- **Rechazado en discovery, no en diseño.**

### Sub-alternativas evaluadas y descartadas

- **DuckDB** en lugar de SQLite. DuckDB excels en analytics (columnar OLAP); para CRUD transaccional típico, SQLite sigue siendo el default. No hay ventaja para este caso.
- **Encriptación at rest** del volumen (LUKS, eCryptfs). Defensa via control de acceso al droplet es suficiente para portfolio scale; añadir cripto desplaza el problema a la gestión de keys sin reducir el riesgo proporcionalmente.
- **`PRAGMA synchronous=FULL`** (default). Considerado para máxima durabilidad; rechazado porque añade fsync por write y la pérdida máxima con NORMAL es ~milisegundos (aceptable para portfolio).

## Referencias

- design doc — D2 (storage decision).
- [ADR 0019](0019-flip-no-persistence.md) — decisión de persistir.
- [ADR 0020](0020-lfpdppp-compliance-architecture.md) — privacy by design en el schema.
- **SQLite WAL:** <https://www.sqlite.org/wal.html>
- **SQLite when to use:** <https://www.sqlite.org/whentouse.html>
- **PostgreSQL bytea anti-pattern:** <https://wiki.postgresql.org/wiki/BinaryFilesInDB>
