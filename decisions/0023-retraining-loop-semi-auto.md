# ADR 0023 — Retraining loop: cron semanal de export + anotación manual + retrain en Colab

**Estado:** Aceptado
**Fecha:** 2026-05-18
**Autor:** Jesús Moreno
**Milestone:** M7

## Contexto

M7 spec D6 define la cadencia de re-entrenamiento del modelo con las imágenes donadas. El operador es single (Jesús), no hay equipo de ML ops. El budget de cómputo es Colab Pay As You Go ($10 destrabó M5 — patrón comprobado).

Tres opciones evaluadas para la cadencia (5a/5b/5c del menú original):

- **5a — Manual batched.** Operador descarga mensualmente, anota offline, retrena en Colab.
- **5b — Semi-auto con cron** (decisión del operador). Cron semanal exporta nuevas filas a un folder de anotación; operador ejecuta el anotado+retrain cuando lo decide.
- **5c — Auto-label con confidence threshold.** El modelo actual auto-anota lo que detecta con confianza alta; bajo confianza va a queue manual.

## Decisión

**Opción 5b — Semi-auto con cron semanal de export + anotación manual + retrain en Colab.**

### Flujo operativo

1. **Cron self-scheduled** en container separado `cron-export` corre cada domingo a las 02:00 UTC.
2. **Export** de las filas con `status = 'pending'` a `/app/contributions/exports/YYYY-WNN/`:
   ```
   exports/2026-W22/
   ├── manifest.csv     # id, sha256, uploaded_at, status, request_id
   ├── images/          # symlinks a /app/contributions/images/{sha256}.jpg
   └── README.md        # cómo descargarlo + anotación + retrain (operador-facing)
   ```
3. **Operador descarga** vía scp manual cuando decide procesar el batch (semanal, mensual — su elección).
4. **Anotación offline** en Roboflow (free tier ≤1000 imgs/proyecto) o LabelImg local. Output COCO format.
5. **COCO → YOLO** vía script existente `scripts/cardd_coco_to_yolo.py` (extendido si necesario).
6. **Merge al training set CarDD** + new batch, retrain en Colab (M5).
7. **Eval contra test split CarDD original** (no incluir contribs nuevos en eval — evita leak).
8. **Verdict deploy** si thresholds D7 del M5 spec se cumplen.
9. **Update pesos** (`rtdetr-cardd.pt`) en volumen Coolify vía scp (mismo patrón M5 T9).
10. **Mark `contributions.status = 'trained'`** para las filas usadas + insert en `training_runs` + `training_contributions`.
11. **Notify subscribers** con `python scripts/notify_contributors.py --model-version=rtdetr-cardd-v2`.

### Implementación técnica

- **Cron self-scheduled** (no crond, no ofelia): el script `export_weekly_batch.py` ejecuta un loop `while True: sleep_until_next_sunday_2am_utc(); run_once()`. Container restart pierde el sleep pero el siguiente domingo ejecuta normal. Simple, sin dependencias.
- **Symlinks en el folder de export** (no copias) para no duplicar bytes.
- **`manifest.csv` con sha256** para que el operador verifique integridad post-scp.
- **README.md por batch** con instrucciones operativas: cómo anotar, qué clase corresponde a qué, qué hacer si la imagen es basura.

## Consecuencias

### Positivas

- **Manual annotation preserva calidad del dataset.** Auto-label con confidence threshold introduce sesgo (el modelo aprende a confirmar lo que ya cree); semi-auto con propagation+revision es estándar de la industria.
- **Verdict gate humano antes del deploy** del nuevo peso. M5 estableció thresholds pre-comprometidos (D7 spec M5); este loop hereda esos thresholds. El operador no deploya basura solo porque sí.
- **Operador en control del momento de retrain.** Cron solo exporta; no consume Colab compute units por sí mismo. El operador decide cuándo gastar (típicamente cuando el batch acumulado justifica el costo).
- **Mismo workflow probado en M5.** El notebook `T7-runbook.md` ya funciona; este loop es repetir M5 con un dataset mergeado.
- **Container `cron-export` separado del backend.** Si el cron tiene un bug y se cae, el backend sigue funcionando. Si el backend cae, el cron sigue exportando (datos quedan en `pending`).

### Negativas

- **Cadencia humana puede saturarse.** Si el batch crece más rápido que la atención del operador (ej. el demo se vuelve viral en LinkedIn), las filas `pending` se acumulan. Mitigación: monitor manual con `SELECT COUNT(*) WHERE status='pending'` mensual.
- **No hay active learning** (el modelo no elige qué samples necesita más). El operador anota lo que llegó, no lo más útil. Trade-off explícito.
- **Anotación humana es lenta** (~30s/img razonable). 100 imgs = 50 min de trabajo. Mitigación: el operador anota el batch que mejor le aporta (skip de imágenes ambiguas o basura).
- **Si el container `cron-export` muere durante el export**, ese batch específico queda incompleto. Mitigación: el script es idempotente — siguiente run reprocesa filas pendientes.

### Neutrales

- **Roboflow free tier** suficiente para portfolio scale (≤1000 imgs/proyecto). Plan B: LabelImg local si Roboflow privacy es issue.
- **El batch semanal puede estar vacío** (sin nuevas contribuciones). El script genera CSV vacío y no falla — exportar nada es válido.
- **`status='trained'` permite analítica** ("cuántas contribuciones han contribuido a un modelo real"). Sirve para LinkedIn posts y para métricas internas del operador.

## Alternativas consideradas

### 5a — Manual batched (sin cron, operador descarga directo)

- Simpler: el operador hace `sqlite3 db.sqlite ".dump"` + `scp images/` cuando decide.
- Rechazado por costo de "qué bajar exactamente" — el cron pre-empaqueta el batch, manifest, README. Reduces friction operativa.

### 5c — Auto-label con confidence threshold

- Modelo actual procesa cada contribución; si `confidence > 0.8` para una clase, auto-anota; si no, manual queue.
- Rechazado: sesgo de confirmación (el modelo refuerza sus propios biases). El per-class de M5 muestra que `scratch` está en mAP 0.51 — auto-anotar lo que el modelo cree son scratches va a reforzar su error sistemático.
- **Reconsiderable** si se establece una métrica de "deriva del modelo" y el operador tiene budget para revisar el queue manual con la cadencia necesaria.

### Cron diario en lugar de semanal

- Innecesariamente granular. El operador no anota diariamente.
- Rechazado.

### Auto-trigger del retrain cuando el batch supera N

- Atractivo en abstracto pero peligroso: el retrain consume Colab Pay As You Go = costo $ real. El operador quiere control humano sobre cuándo gastar.
- Rechazado para M7. Reconsiderable para M8+ si el ciclo se vuelve predecible.

### Annotation con tools custom (ej. Label Studio self-hosted)

- Más control + más overhead. Roboflow free + LabelImg local cubren el caso.
- Rechazado por costo operativo.

### Eval incluyendo contribuciones nuevas en test split

- Tentador (más data → mejor eval).
- Rechazado: leak inevitable. El test split CarDD original es el invariante para comparar epochs/runs. Contribuciones nuevas entran solo a train.

## Referencias

- design doc — D6 (retraining cadence).
- design doc — D7 thresholds verdict (heredados).
- [ADR 0017](0017-finetune-cardd-result.md) — verdict deploy del primer fine-tune (template para los siguientes).
- **Roboflow:** <https://roboflow.com>
- **LabelImg:** <https://github.com/HumanSignal/labelImg>
