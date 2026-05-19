# ADR 0017 — Resultado del fine-tune RT-DETRv2 sobre CarDD: deploy

**Estado:** Aceptado
**Fecha:** 2026-05-06
**Autor:** Jesús Moreno

## Contexto

[ADR 0001](0001-pretrained-first.md) estableció en M0 que el pipeline corre con pesos preentrenados hasta M5, donde se fine-tunea RT-DETRv2 sobre CarDD y se decide si los pesos custom reemplazan a los preentrenados en producción. El spec M5 cerró con D7: thresholds **pre-comprometidos antes de ver números** para que el verdict sea defendible regardless del outcome:

| Resultado | Acción |
|---|---|
| mAP@0.5 ≥ 0.40 ∧ latency ≤ 1.2× baseline | **`deploy`** |
| 0.20 ≤ mAP@0.5 < 0.40 | `publish_no_deploy` (quedarse con pretrained-mock) |
| mAP@0.5 < 0.20 ∨ latency > 1.2× | `publish_with_diagnosis` (quedarse con pretrained-mock + diagnosticar) |

Este ADR documenta el resultado real del experimento y la decisión derivada.

## Decisión

**Verdict: `deploy`.** Los pesos `models/rtdetr-cardd.pt` reemplazan al pretrained `rtdetr-l.pt` en producción vía la env var `DETECTOR_WEIGHTS` configurada en Coolify (T9, sec D6 del spec M5).

Métricas del eval ([JSON crudo](../metrics/baseline-vs-finetuned.json), [narrativa](../metrics/baseline-vs-finetuned.md)):

| Criterio D7 | Threshold | Real | Pasa |
|---|---|---|---|
| mAP@0.5 (test split) | ≥ 0.40 | **0.701** | ✓ |
| Latency ratio (CPU warm) | ≤ 1.2× | **1.057×** | ✓ |

Per-class mAP@0.5: `broken_glass` 0.966, `broken_lamp` 0.819, `flat_tire` 0.799, `dent` 0.591, `crack` 0.523, `scratch` 0.509. Detalle en [`baseline-vs-finetuned.md`](../metrics/baseline-vs-finetuned.md#per-class).

## Consecuencias

### Positivas

- **Categorías reales del dominio.** Las clases predichas por el detector ya son las del enum `DamageType` (`dent`, `scratch`, etc.) — no proxies COCO traducidos por `coco_to_damage_type` (placeholder M2). El [ADR 0008](0008-rtdetr-vs-yolo.md) anticipó este reemplazo; D8 del spec M5 dejó el detector listo para servirlo sin más cambios runtime (commit [`2f8c8b8`](https://github.com/Jesusmcyc/car-crash-analyzer/commit/2f8c8b8)).
- **mAP agregado en territorio production-grade.** 0.701 supera por amplio margen el floor de 0.40 que el spec definió como "se siente útil en demo". La probabilidad de que una imagen de demo aleatoria muestre detecciones correctas es alta.
- **Latencia preservada.** El delta de 32 ms (5.7%) es despreciable para el target del demo (~3 s warm). El backbone es el mismo que el pretrained; solo cambian los pesos.
- **El roadmap del proyecto se cumplió.** [ADR 0001](0001-pretrained-first.md) prometió "el fallback obvio: si el fine-tune no mejora la baseline preentrenada, el sistema funciona igual con los pesos originales". El fine-tune sí mejoró, por lo tanto se despliega — pero el rollback documentado en spec D6 (borrar la env var, redeploy, queda pretrained-mock) sigue disponible si algo se descubre en post-deploy.

### Negativas

- **Brecha per-class entre clases con bordes vs. textura.** `broken_glass`/`broken_lamp`/`flat_tire` están en mAP ≥ 0.80; `dent`/`crack`/`scratch` en 0.50-0.59. Esto no invalida el verdict — el agregado supera el threshold y el comportamiento es coherente con la naturaleza de los daños — pero el demo presenta tres tiers de confianza implícitos que el frontend debería visibilizar (caveat al usuario cuando `confidence < 0.6`).
- **`flat_tire` 0.799 tiene varianza estadística alta.** Solo 32 instancias en test split (la clase con menos samples). El intervalo de confianza informal está alrededor de ±0.10 — un test split aleatorio diferente podría mostrar mAP de 0.65-0.85. La narrativa del demo debe comunicar "≥ 0.70 en test" en lugar de citar el 0.799 como número definitivo. Detalle en [`baseline-vs-finetuned.md` sec Per-class](../metrics/baseline-vs-finetuned.md#per-class).
- **No medimos overfitting cuantitativamente.** El reporte solo incluye test split. Val split (810 imgs, mAP@0.5 = 0.700 según el último epoch del training) está implícito en los plots de Ultralytics dentro de `rtdetr_cardd_run1/` en Drive. El hecho de que test (0.701) y val (0.700) sean prácticamente idénticos sugiere generalización buena, pero no se commiteó como métrica formal.
- **Wall-clock de training real subestimado por el spec.** Spec D3 estimó "~2-3 h en Colab T4". El tiempo de `model.train()` puro fue **1.234 h (74 min)** según `results.csv`, pero el wall-clock total con setup overhead × 2 sesiones (clone, pip, drive mount, label scan) y el gap del disconnect estuvo en **~2-3 h**. El segundo intento requirió pagar Pay As You Go ($10 USD = 100 compute units, 8 consumed) para destrabar la cuota.
### Neutrales / followups

- **Class weights no aplicados** (decisión D3-diferida del spec). La siguiente iteración M5.5 podría añadirlos si se observa que `scratch`/`crack` son frecuentes en uso real y sus mAPs limitan la utilidad. No bloquea T9.
- **`APP_VERSION` en Coolify aún sin inyectar** (heredado de M4). El smoke post-deploy de T9 acepta `model_version: "rtdetr-cardd-unknown"` como verde provisorio per la cláusula del spec (sec 348). Fix de 1 línea en la UI de Coolify, no bloquea.

## Alternativas consideradas

1. **Quedarse con pretrained-mock COCO + `coco_to_damage_type` (verdict `publish_no_deploy`).** Rechazada — el threshold D7 (mAP@0.5 ≥ 0.40) se superó con holgura. Mantener pretrained sería honestidad sobre un dato falso (el pretrained-mock no reporta su mAP real porque no tiene clases CarDD; D7 ya cubre ese argumento).
2. **Esperar a una segunda corrida con class weights antes de desplegar.** Rechazada — el verdict actual ya cumple criterios deploy. Una corrida con class weights es valiosa como M5.5 pero retrasarse 1-2 días más en deployar el peso ya entrenado no aporta — el peso actual genera valor de demo desde el momento que aterriza en Coolify.
3. **Entrenar más epochs (extender `epochs=50` a 100, `patience=20`).** Rechazada — Ultralytics disparó early-stop en epoch 40 con 10 epochs sin mejora desde epoch 30. Más epochs habrían sido tiempo y compute desperdiciado. El best.pt de epoch 30 captura el peak.
4. **Backbone más grande (RT-DETR-x en lugar de -l).** Rechazada implícitamente por D3 del spec — el demo CPU-only del runtime de producción ya está cerca del techo de latencia tolerable con -l. -x agregaría ~50% latencia para una mejora estimada del orden de mAP +0.02-0.05; trade-off no justificado para un demo.

## Referencias

- [ADR 0001](0001-pretrained-first.md) — pretrained-first; este ADR cierra esa promesa.
- [ADR 0008](0008-rtdetr-vs-yolo.md) — placeholder COCO→DamageType; obsoleto en uso pero `coco_to_damage_type` se mantiene como fallback (D8 spec M5).
- spec M5 — D3 (hyperparams), D4 (eval), D6 (deploy), D7 (verdict thresholds).
- [`Docs/metrics/baseline-vs-finetuned.json`](../metrics/baseline-vs-finetuned.json) — números crudos.
- [`Docs/metrics/baseline-vs-finetuned.md`](../metrics/baseline-vs-finetuned.md) — interpretación per-class y desviaciones operativas.
