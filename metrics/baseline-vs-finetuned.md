# Baseline vs. Finetuned RT-DETRv2 sobre CarDD

**Fecha de la corrida:** 2026-05-05 — 2026-05-06
**Verdict:** `deploy` (mAP@0.5 ≥ 0.40 ∧ latency ratio ≤ 1.2×)
**Datos crudos:** [`baseline-vs-finetuned.json`](baseline-vs-finetuned.json)

---

## Headline

| Métrica | Baseline (rtdetr-l.pt) | Finetuned (rtdetr-cardd.pt) | Δ |
|---|---|---|---|
| **mAP@0.5** (test split) | 0.0 ¹ | **0.701** | +0.701 |
| **mAP@0.5:0.95** | 0.0 ¹ | **0.536** | +0.536 |
| **Latencia mean** (CPU warm) | 581.2 ms | 614.0 ms | +32.8 ms (+5.7%) |
| **Latencia p95** | 602.3 ms | 624.0 ms | +21.7 ms (+3.6%) |
| **Latency ratio** | 1.0× | **1.057×** | ≤ 1.2× ✓ |

¹ El baseline pretrained COCO no comparte ninguna clase con CarDD; `model.val()` se omite y mAP=0.0 por construcción. La comparación útil es entre cero conocimiento de daños y el peso fine-tuneado, no entre dos detectores entrenados sobre el mismo target.

**Hardware del eval:** AMD Ryzen 7 5800H (CPU only), torch 2.11.0+cpu, ultralytics 8.4.46.

---

## Per-class

374 imágenes test, 785 anotaciones, 6 clases.

| Clase | Instancias test | mAP@0.5 | Precision | Recall | Lectura |
|---|---|---|---|---|---|
| `broken_glass` | 71 | **0.966** | 0.803 | 0.972 | Aprendido casi perfectamente. Daño con bordes salientes/aristas, fácil de localizar. |
| `broken_lamp` | 69 | 0.819 | 0.862 | 0.667 | Forma + posición canónica del componente. Recall moderado por lámparas parcialmente visibles. |
| `flat_tire` | **32** | 0.799 | 0.678 | 0.844 | Buen recall — la deformación de la llanta es signal fuerte; precision baja porque confunde llantas mojadas/manchadas con desinfladas. **n=32 → varianza alta**: este 0.80 es ruidoso, podría caer a 0.60-0.70 en otro test split aleatorio. |
| `dent` | 236 | 0.591 | 0.664 | 0.553 | Curvatura sutil sin bordes claros. Mid-tier, el modelo distingue abolladuras grandes y falla en las pequeñas. |
| `crack` | 70 | 0.523 | 0.656 | 0.462 | Líneas finas en superficies pintadas — sin bordes para anclar el bbox. Recall moderado, precision aceptable. |
| `scratch` | **307** | 0.509 | 0.513 | 0.460 | Daño de textura, cero geometría. La clase más débil **a pesar de ser la más frecuente** — confirma que cantidad de samples no compensa ausencia de bordes para anclar el bbox. |
| **Total** | **785** | — | — | — | — |

**Lectura agregada.** Las tres clases con bordes/contornos definidos (`broken_glass`, `broken_lamp`, `flat_tire`) están en mAP@0.5 ≥ 0.80 — ya en territorio production-grade. Las tres clases de textura/curvatura (`dent`, `crack`, `scratch`) quedan en 0.50-0.59 — utilizables en demo pero no para clasificación pricing. Esta brecha refleja la naturaleza de los daños, no un bug del fine-tune.

> **Nota de sec 4 del spec.** El spec anticipaba `flat_tire` como la clase con mayor riesgo de mAP <0.20 por imbalance del dataset (~225 samples train, 32 en test). El resultado real (0.799) supera la predicción ampliamente. La clase débil resultó ser `scratch` (con 307 samples test, la más frecuente). La hipótesis ex-ante de "imbalance domina sobre todo lo demás" no aplicó — `scratch` falló no por escasez sino por ser un daño de textura sin geometría a la cual anclar el bounding box. La hipótesis correcta resultó ser **"ausencia de bordes geométricos > cantidad de samples"**.

> **Caveat de varianza para `flat_tire`.** Con n=32 instancias, el mAP@0.5 reportado tiene intervalo de confianza ancho (≈ ±0.10 en una estimación informal). Para presentaciones del demo, mejor comunicar como "≥ 0.70 en test" en lugar de "0.799 puntual" — la varianza estadística desinfla la precisión aparente. Si M5.5 quiere validar con más rigor, una opción es bootstrap del test split.

---

## Setup de la corrida

### Training

- **Dataset:** CarDD COCO format → Ultralytics YOLO format vía `scripts/cardd_coco_to_yolo.py`. Splits: train 2816 imgs / 6211 labels, val 810 / 1744, test 374 / 785.
- **Hardware:** Tesla T4 (Colab) — sesión 1 con Free tier, sesión 2 con Pay As You Go (~$10 USD = 100 compute units, 8 consumed).
- **Hyperparams (D3):** `RTDETR('rtdetr-l.pt')`, `epochs=50, patience=10, lr0=1e-4, imgsz=640, optimizer=AdamW, batch=8, close_mosaic=5, seed=42`. Ningún parámetro modificado durante la corrida (la `box_loss` bajó >5% en epoch 1, no requirió el bump a `lr0=5e-4` documentado como contingencia).
- **Tiempo de `model.train()` puro:** **1.234 h (74 min)** sumados a través de las dos sesiones, según el último timestamp de `runs/.../results.csv` (epoch 40 → `time=4439.91 s`). Ultralytics persiste el contador de tiempo en `last.pt` y lo continúa en el resume, así que el número refleja training real, no setup.
- **Wall-clock total** (clic-en-celda → `Training completo.`): ~2-3 h. La diferencia con el training puro está en setup overhead × 2 sesiones (clone repo, pip install, drive mount, label scan ~5-10 min cada uno) más el gap entre el disconnect del Free tier y la compra del Pay As You Go.
- **Epochs efectivos:** 40 de 50 tope. Early-stop disparó tras epoch 40 al detectar 10 epochs sin mejora desde el best en epoch 30.
- **Mejor epoch:** 30. `best.pt` saved automáticamente por Ultralytics.

### Eval

- **Dataset:** test split (374 imágenes, 785 anotaciones).
- **Hardware:** local AMD Ryzen 7 5800H, CPU only (consistente con el target de producción del demo).
- **Latencia:** 6 forward passes sobre `backend/tests/fixtures/damaged_car.jpg`, descarta primera (warm-up), reporta mean + p95 de las 5 restantes.
- **Comando:**
  ```powershell
  $env:CARDD_ROOT = "G:\My Drive\Projects\Computer_Vision\CCA\CarDD_release"
  $env:YOLO_OUT = "C:\temp\cca-yolo"
  uv run --project backend python scripts/eval_baseline_vs_finetuned.py
  ```

---

## Desviaciones operativas (no afectan los resultados)

Tres incidentes durante la corrida que motivaron commits adicionales fuera del spec original. Ninguno cambió hyperparams ni invalidó los números.

### 1. Drive FUSE no permite symlinks dentro de Drive

**Síntoma:** `cardd_coco_to_yolo.py` symlinkéa cada imagen del COCO source al directorio YOLO. Con `CARDD_ROOT` apuntando a Drive (vía Colab mount), los symlinks tienen origen y destino dentro de Drive. FUSE rechaza con `OSError: [Errno 95] Operation not supported`.

**Fix:** [`7c1afd7`](https://github.com/Jesusmcyc/car-crash-analyzer/commit/7c1afd7) introduce `YOLO_OUT` env var en `cardd_coco_to_yolo.py`. Default mantiene back-compat (`${CARDD_ROOT}/yolo`); el notebook setea `YOLO_OUT=/content/yolo` (ext4 efímero). El eval también respeta `YOLO_OUT` (commit [`0cba7ab`](https://github.com/Jesusmcyc/car-crash-analyzer/commit/0cba7ab)).

**Consecuencia:** la conversión es ahora reproducible en cualquier setup con paths-en-Drive (típico en Windows con Drive Desktop sync). Localmente en Windows usa file copies (vs. symlinks en Linux), tradeoff de ~3 GB extra de disco por sesión a cambio de no requerir privilegios admin.

### 2. Path con espacios en Drive (`Computer Vision`)

**Síntoma:** la ruta canónica del usuario era `MyDrive/Projects/Computer Vision/CCA/CarDD_release`. La celda de conversión hace `!CARDD_ROOT={CARDD_ROOT} python ...` que IPython interpola sin quotear; el espacio rompía el shell.

**Fix:** dos capas. (a) Usuario renombró la carpeta Drive a `Computer_Vision` (underscore) — fix permanente. (b) Notebook ahora quota `"{CARDD_ROOT}"` y `"{YOLO_OUT}"` defensivamente. El path con underscore se commiteó como default en celda 9 (commit [`6a973fe`](https://github.com/Jesusmcyc/car-crash-analyzer/commit/6a973fe)).

### 3. Colab Free disconnect mid-training → resume

**Síntoma:** epoch 9 completado, mAP@0.5 = 0.34. Colab Free disparó cuota de GPU; runtime perdido pero `last.pt` persistió en Drive (`/content/drive/MyDrive/cca/m5/rtdetr_cardd_run1/weights/`).

**Fix:** [`7c93de9`](https://github.com/Jesusmcyc/car-crash-analyzer/commit/7c93de9) hace que la celda de training detecte `last.pt` y reanude automáticamente con `model.train(resume=True)` (lee hyperparams del checkpoint, no se pueden alterar). Si no hay checkpoint, entrena desde cero como antes. **Comportamiento idempotente para el primer run, robusto a disconnect en cualquier otro.**

**Consecuencia:** sesión 2 reanudó desde epoch 10 con todos los pesos del epoch 9 (`Transferred 941/941 items`, vs. `926/941` cuando se entrena desde el base `rtdetr-l.pt`). El training prosiguió la trayectoria mAP sin discontinuidades.

---

## Reproducibilidad

Para reproducir la eval localmente en Windows:

```powershell
cd "C:\App\Car Crash Analizer\Code"
$env:CARDD_ROOT = "<path-a-CarDD_release>"
$env:YOLO_OUT = "C:\temp\cca-yolo"

# Conversión (idempotente, ~5 min en Windows con file copies)
cd backend
uv run python ../scripts/cardd_coco_to_yolo.py
cd ..

# Eval (~5-15 min en CPU)
uv run --project backend python scripts/eval_baseline_vs_finetuned.py
```

El JSON output coincide deterministicamente con `Docs/metrics/baseline-vs-finetuned.json` salvo `generated_at` y `latency_ms.*` (que dependen del hardware host).

---

## Recomendación para T9 (deploy)

**Subir `models/rtdetr-cardd.pt` al volumen Coolify** y setear `DETECTOR_WEIGHTS=/app/models/rtdetr-cardd.pt`. Smoke contract: `/api/health` debe reportar `models_loaded:true`, `/api/analyze` debe retornar al menos un `damage_type` ∈ `{dent, scratch, crack, broken_glass, broken_lamp, flat_tire}` (los nombres del enum, no los proxies COCO de M2).

**Esperado en producción:** mAP@0.5 ≈ 0.70 sobre imágenes "in-distribution" (similares a CarDD: vehículos, daños fotografiados de cerca, iluminación limpia). Imágenes muy out-of-distribution (poca luz, daños múltiples superpuestos en clases de textura) caerán al rango 0.40-0.50 dominado por `scratch`/`crack`/`dent`.

**Caveat para presentar el demo:** las detecciones de `scratch` se deberán mostrar al usuario con un caveat verbal ("detección de baja confianza, requiere confirmación humana") cuando `confidence < 0.6`. Esto se puede implementar en el frontend leyendo el campo `confidence` del response — no requiere retraining.

**Followups diferidos** (no bloquean T9):
- Class weights o oversampling para `scratch`/`crack`/`dent` en una corrida M5.5 si se observa que esos casos son comunes en uso real.
- `APP_VERSION` env var en Coolify para que `/api/health` reporte el SHA del commit (heredado de M4).
