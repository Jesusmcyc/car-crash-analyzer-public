# ADR 0019 — Flip "no persistencia" para enabled continued training

**Estado:** Aceptado
**Fecha:** 2026-05-18
**Autor:** Jesús Moreno
**Milestone:** M7

## Contexto

`architecture doc` sec 11 listaba desde M0 entre los items out-of-scope:

> - Persistencia (DB). Las inferencias son fire-and-forget.

Esa restricción mantuvo el backend stateless durante M0–M6: el endpoint `/api/analyze` procesa la imagen en memoria, devuelve el reporte JSON, y libera todos los recursos. Cero archivos persistidos, cero DB. Fue una decisión correcta para los milestones de modelado (M0–M5) y entrega de portfolio (M6).

M7 introduce la feature de **continued training** sobre imágenes donadas voluntariamente por los visitantes del demo público. Las decisiones M7 (spec D2 + D3) requieren persistir el subset de imágenes con consentimiento expreso (LFPDPPP), junto con metadata (`sha256`, `uploaded_at`, `consent_version`, `subscriber_id` opcional) y el email del donante para notificación post-training (D7).

La restricción "no persistencia" del scope original debe levantarse de forma controlada para este subset, sin abrir la puerta a persistir todas las inferencias.

## Decisión

**La restricción "no persistencia" se levanta SOLO para imágenes con `consent=true` explícito.** El resto del pipeline sigue stateless como hasta hoy:

| Caso | Persistencia | Justificación |
|---|---|---|
| `/api/analyze` con `consent=false` (o sin el campo) | **No** | Backward compatible M5; fire-and-forget |
| `/api/analyze` con `consent=true` y consent_version vigente | **Sí** | LFPDPPP — consentimiento expreso |
| Logs estructurados de inferencia | No persistidos en DB | Loguru stdout, captura container-level |
| Métricas operacionales agregadas | No persistidas | Out of scope M7 |

**Stack de persistencia (detallado en ADR 0021):**

- **SQLite** single-file (`/app/contributions/db.sqlite`), WAL mode.
- **Volumen Docker** `contributions-images` montado en `/app/contributions/` (Coolify-orquestado, mismo patrón que `finetuned-models` de M5 T9).
- **Sin Postgres, sin S3, sin Redis** — overkill para portfolio scale.

**Scope original actualizado:** se mantiene el bullet "Persistencia (DB)" tachado en `architecture.md` con redirect explícito a este ADR. El operador que llegue cold al repo entiende el matiz desde la primera lectura.

## Consecuencias

### Positivas

- **Continued training se vuelve viable.** Sin persistencia el ciclo de mejora del modelo dependía 100% del dataset CarDD original (4k imgs); con persistencia el ciclo se extiende a la comunidad del demo.
- **Demo público gana feature diferenciable para portfolio.** "Una app que aprende de quien la usa" es highlight diferenciable en LinkedIn vs. otra demo CV genérica.
- **Engagement narrativo para LinkedIn posts.** Cada nueva versión del modelo da material publicable: "v2 entrenó con N fotos donadas por la comunidad".
- **Disciplina operativa demostrable.** Implementar consent, ARCO rights, EXIF stripping, withdrawal endpoint, retraining loop completo es trabajo "production-grade" que un revisor técnico senior reconoce — más fuerte que añadir más slides al deck.

### Negativas

- **Nueva superficie legal (LFPDPPP).** Detalle en [ADR 0020](0020-lfpdppp-compliance-architecture.md). Si el aviso de privacidad es insuficiente o el flujo ARCO falla, el riesgo es real (reclamo ante INAI, autoridad mexicana).
- **Backup operacional manual.** El volumen Coolify sobrevive container restart pero el operador debe descargar `db.sqlite` + `images/` mensualmente al filesystem propio. Backup automatizado queda fuera de scope M7 (potencial M7.5).
- **Disco del droplet limitado.** ~5 GB esperados para 10k contribs (resize + JPEG q=85 = ~500 KB/img). Monitor manual mensual con `df -h`.
- **Una vía de fallo más:** SQLite WAL puede corromperse si Coolify mata el container durante un write. Mitigación: `PRAGMA synchronous=NORMAL` + WAL mode; pérdida máxima estimada en milisegundos.

### Neutrales

- **El scope original se actualiza con redirect a este ADR** — bullet "Persistencia (DB)" en `architecture.md` queda tachado con nota inline. El operador cold lee el matiz desde la primera pasada.
- **El resto de items "out of scope" sigue vigente** — auth, i18n, mobile-first, PWA, webhooks. M7 levanta exactamente un bullet, no abre paréntesis a otros.
- **Pipeline CV (`backend/app/pipeline/`) no se toca.** La feature M7 vive en módulo nuevo adyacente `backend/app/contributions/`. Separación clara para que el revisor entienda qué es modelado y qué es data collection.

## Alternativas consideradas

1. **Postgres + S3-compatible (DigitalOcean Spaces $5/mes).** Rechazado — overkill para portfolio scale. SQLite WAL + volumen Docker suficiente para los primeros 10k contribs estimados. Reconsiderar si el volumen supera 50 GB o si se necesita acceso concurrente desde múltiples writers (no aplica para single-operator).
2. **Postergar la feature a M7.5 después de UI polish.** Rechazado — contradice la dirección portfolio (memoria `project-portfolio-direction`); el polish + paper se integran mejor con la feature ya activa que separando los milestones.
3. **Persistir TODAS las inferencias (incluso sin consent) anonimizadas.** Rechazado — LFPDPPP exige consentimiento expreso para el tratamiento; "anonimización" sin consent del titular no aplica como base legal sólida en el régimen mexicano cuando el dato puede ser potencialmente identificable (imágenes con placas/rostros).
4. **Levantar la restricción "no persistencia" sin restringir el caso de uso.** Rechazado por principio de minimización — la decisión M7 es específica (continued training, opt-in expreso), no genérica.

## Referencias

- `architecture doc` sec 11 — restricción original (modificada en M7).
- design doc — spec M7 D1-D3, D9.
- [ADR 0020](0020-lfpdppp-compliance-architecture.md) — cumplimiento legal asociado.
- [ADR 0021](0021-storage-sqlite-volume.md) — detalle técnico de SQLite + volumen.
- [ADR 0022](0022-email-resend-transactional.md) — manejo de email separado.
- [ADR 0023](0023-retraining-loop-semi-auto.md) — pipeline de re-entrenamiento.
- LFPDPPP: <https://www.diputados.gob.mx/LeyesBiblio/pdf/LFPDPPP.pdf>
