# ADR 0011 — Heurística de severidad explicable

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno
**Milestone:** M3
**Spec asociado:** design doc sec D4

## Contexto

CarDD no incluye anotaciones de severidad real (LEVE/MODERADO/SEVERO);
únicamente clases de daño. Sin datos supervisados, un modelo aprendido para
severidad sería ruido — el spec del proyecto exige que la
severidad sea heurística explicable, no modelo aprendido.

Tres dimensiones intuitivas para combinar:
- **Área relativa** del daño respecto al vehículo.
- **Tipo de daño** (vidrio roto > grieta > abolladura > rayón).
- **Posición** (chasis principal vs cosmético).

## Decisión

Score multiplicativo:

```
score = area_ratio × type_weight × position_weight
```

con:

| DamageType | type_weight |
|---|---|
| BROKEN_GLASS | 1.0 |
| CRACK | 0.7 |
| BROKEN_LAMP | 0.6 |
| DENT | 0.5 |
| FLAT_TIRE | 0.4 |
| SCRATCH | 0.3 |

`position_weight = 1.15` si el centroide del bbox cae en `0.30 ≤ y_norm ≤ 0.70`
(zona vertical de chasis principal); `1.0` en otro caso.

`area_ratio = mask.area_px / vehicle_bbox.area_px`, capado a `[0, 1]`. Si no
hay vehicle_bbox dominante, fallback a image_dims.

Mapeo a Severity por umbrales sobre el score:
- `score < 0.04` → LEVE
- `0.04 ≤ score < 0.12` → MODERADO
- `score ≥ 0.12` → SEVERO

`severity_factors` del response contiene exactamente las llaves
`{area_ratio, type_weight, position_weight}` — el frontend M1 ya las renderiza.

**Cost ranges:** mantener placeholders M1 (LEVE 2k–8k, MODERADO 8k–25k,
SEVERO 25k–80k MXN). Documentados como "demo, no estudio actuarial" en
`reporter.py`. Calibrar requiere datos actuariales fuera de scope.

**Fallback bbox-polygon:** cuando SAM 2 rechaza la máscara (D3), el `Damage`
mantiene el contrato `Mask.polygon ≥ 3` con un polígono derivado del bbox.
La severidad usa `area_ratio=0` vía `estimate_damage(mask=None)`.

## Consecuencias

### Positivas

- **Multiplicativo captura intuición humana.** Un vidrio roto pequeño en
  chasis (8% × 1.0 × 1.15 = 0.092 MODERADO) vs un rayón grande fuera de chasis
  (30% × 0.3 × 1.0 = 0.09 MODERADO también) → calibración razonable.
- **Auditable.** Las constantes (`_TYPE_WEIGHTS`, `_POSITION_BAND`,
  `_SEVERITY_THRESHOLDS`) son públicas en `severity.py`. Un revisor externo
  puede contrastar sin abrir este ADR.
- **Frontend cero churn.** Llaves canónicas de severity_factors mantienen
  compatibilidad con M1.

### Negativas

- **Umbrales calibrados a ojo.** No hay validación contra datos reales — el
  ADR es honesto al respecto. Si se necesita evaluación cuantitativa,
  requiere sesgar tiempo a etiquetar fixtures con severidad ground truth, lo
  cual está fuera de scope del demo.
- **Position_weight binario.** Una banda dura (`0.30–0.70`) puede dar saltos
  visibles en daños limítrofes. Mitigación posible (no implementada): sigmoid
  suave centrado en 0.5. No se hace ahora porque añade complejidad sin
  observación de problema.
- **Cost ranges no calibrados.** Documentado como placeholder. Aceptable
  para demo de portfolio.

## Alternativas consideradas

### Score aditivo (suma ponderada)

Rechazado. Un rayón cosmético del 30% obtiene severidad similar a un vidrio
roto del 8% si los pesos se suman — la intuición humana es claramente
multiplicativa: daños grandes en zona crítica son exponencialmente peores.

### Umbrales por área cruda (handoff M3 propuesta original)

Rechazado. Clasificar por `area < 5%` → LEVE elimina la influencia del
type_weight en el resultado final, anulando el matiz vidrio-roto vs rayón.

### Modelo aprendido (regression sobre severity)

Rechazado por falta de ground truth. Re-evaluable si en M5+ se obtiene un
dataset etiquetado en severidad.

## Referencias

- Spec maestro sec M3 — exit criterion de severidad explicable.
- ADR 0001 — pretrained-first (no aprender severidad sin datos).
- ADR 0010 — sam2-zero-shot (provee la mask que entra como input).
- ADR 0012 — mask-encoding-polygon-cap (define cuándo el mask es None).
