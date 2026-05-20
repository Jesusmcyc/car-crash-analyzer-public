# ADR 0011 — Explainable severity heuristic

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno
**Milestone:** M3
**Related spec:** design doc sec D4

## Context

CarDD does not include real severity annotations (LEVE/MODERADO/SEVERO);
only damage classes. Without supervised data, a learned model for
severity would be noise — the project spec requires that
severity be an explainable heuristic, not a learned model.

Three intuitive dimensions to combine:
- **Relative area** of the damage with respect to the vehicle.
- **Damage type** (broken glass > crack > dent > scratch).
- **Position** (main chassis vs cosmetic).

## Decision

A multiplicative score:

```
score = area_ratio × type_weight × position_weight
```

with:

| DamageType | type_weight |
|---|---|
| BROKEN_GLASS | 1.0 |
| CRACK | 0.7 |
| BROKEN_LAMP | 0.6 |
| DENT | 0.5 |
| FLAT_TIRE | 0.4 |
| SCRATCH | 0.3 |

`position_weight = 1.15` if the bbox centroid falls within `0.30 ≤ y_norm ≤ 0.70`
(the vertical band of the main chassis); `1.0` otherwise.

`area_ratio = mask.area_px / vehicle_bbox.area_px`, capped to `[0, 1]`. If there
is no dominant vehicle_bbox, it falls back to image_dims.

Mapping to Severity via thresholds on the score:
- `score < 0.04` → LEVE
- `0.04 ≤ score < 0.12` → MODERADO
- `score ≥ 0.12` → SEVERO

The response's `severity_factors` contains exactly the keys
`{area_ratio, type_weight, position_weight}` — the M1 frontend already renders them.

**Cost ranges:** keep the M1 placeholders (LEVE 2k–8k, MODERADO 8k–25k,
SEVERO 25k–80k MXN). Documented as "demo, not an actuarial study" in
`reporter.py`. Calibrating them requires actuarial data, which is out of scope.

**Bbox-polygon fallback:** when SAM 2 rejects the mask (D3), the `Damage`
keeps the `Mask.polygon ≥ 3` contract with a polygon derived from the bbox.
Severity then uses `area_ratio=0` via `estimate_damage(mask=None)`.

## Consequences

### Positives

- **The multiplicative form captures human intuition.** A small broken-glass area on
  the chassis (8% × 1.0 × 1.15 = 0.092, moderate) vs a large scratch off the chassis
  (30% × 0.3 × 1.0 = 0.09, also moderate) → reasonable calibration.
- **Auditable.** The constants (`_TYPE_WEIGHTS`, `_POSITION_BAND`,
  `_SEVERITY_THRESHOLDS`) are public in `severity.py`. An external reviewer
  can cross-check them without opening this ADR.
- **Zero frontend churn.** The canonical severity_factors keys keep
  compatibility with M1.

### Negatives

- **Eyeballed thresholds.** There is no validation against real data — the
  ADR is honest about it. If quantitative evaluation is needed, it
  requires diverting time to label fixtures with ground-truth severity, which
  is out of scope for the demo.
- **A binary position_weight.** A hard band (`0.30–0.70`) can produce visible
  jumps for borderline damage. A possible mitigation (not implemented): a smooth
  sigmoid centered at 0.5. Not done now because it adds complexity with no
  observed problem.
- **Uncalibrated cost ranges.** Documented as a placeholder. Acceptable
  for a portfolio demo.

## Alternatives considered

### Additive score (weighted sum)

Rejected. A 30% cosmetic scratch gets a severity similar to 8% broken
glass if the weights are summed — human intuition is clearly
multiplicative: large damage in a critical zone is exponentially worse.

### Thresholds on raw area (original M3 handoff proposal)

Rejected. Classifying by `area < 5%` → LEVE removes the influence of
type_weight on the final result, nullifying the broken-glass vs scratch nuance.

### Learned model (regression on severity)

Rejected for lack of ground truth. Re-evaluable if an M5+ stage obtains a
dataset labeled for severity.

## References

- Master spec sec M3 — exit criterion for explainable severity.
- ADR 0001 — pretrained-first (do not learn severity without data).
- ADR 0010 — sam2-zero-shot (provides the mask that comes in as input).
- ADR 0012 — mask-encoding-polygon-cap (defines when the mask is None).
