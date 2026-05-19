# ADR 0010 — SAM 2 zero-shot, hiera-tiny variante

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno
**Milestone:** M3
**Spec asociado:** design doc sec D1

## Contexto

M3 cierra el pipeline con segmentación real. El spec maestro (sec M3) definió
SAM 2 como segmentador y dejó la decisión zero-shot vs fine-tune para este ADR.

CarDD provee ~4k imágenes con anotaciones de daños en formato COCO. SAM 2 fue
entrenado con ~11M máscaras (SA-V dataset). Tres opciones reales:

1. **Zero-shot, hiera-tiny** (~40 MB, ~1–2 s/máscara CPU).
2. **Zero-shot, hiera-base+** (~160 MB, ~3–4 s/máscara CPU).
3. **Fine-tune con CarDD.**

## Decisión

**Zero-shot con `facebook/sam2-hiera-tiny`** como default. Path de upgrade a
`facebook/sam2-hiera-base-plus` por env var `SEGMENTER_MODEL` documentado.

## Consecuencias

### Positivas

- Latencia compatible con presupuesto M3 (~9–12 s pipeline completo CPU sobre
  3 daños típicos). Hiera-base+ tocaría 18–22 s y rebasaría `proxy_read_timeout`.
- Sin pipeline de entrenamiento adicional. M5 entrena el detector; el segmenter
  no entra en ese ciclo.
- Calidad zero-shot suficiente para el demo. Las diferencias mIoU tiny vs
  base+ rondan 2–3 puntos sobre SA-V; con simplificación Douglas–Peucker
  aplicada en M3 (ADR 0012), la diferencia visible se diluye.

### Negativas

- Calidad de máscara potencialmente inferior a un modelo fine-tuned con CarDD.
  Mitigación: el demo prioriza pipeline shape-completo sobre pixel-perfect;
  M5 puede revisitar si una insurtech real requiere mayor fidelidad.
- Costos de inferencia escalan lineal con N daños. Mitigación: el detector ya
  filtra a vehículos COCO; N típico ≤5.

## Alternativas consideradas

### Fine-tune SAM 2 con CarDD

Rechazado. CarDD tiene tres órdenes de magnitud menos máscaras que SA-V;
fine-tune sin disciplina de regularización degrada significativamente
la generalización. El comportamiento está documentado en literatura de SAM 1
(catastrophic forgetting).

### SAM 1 (`facebook/sam-vit-base`)

Rechazado. SAM 2 es estrictamente superior en SA-V eval; la API en
`transformers` es equivalente. Sin razón técnica para volver atrás.

### Hiera-base+ default

Rechazado por presupuesto de latencia (ver sec Consecuencias). Disponible
como opt-in vía `SEGMENTER_MODEL=facebook/sam2-hiera-base-plus` para casos
donde el operador acepte 2× latencia.

## Referencias

- SAM 2 oficial: <https://github.com/facebookresearch/sam2>
- Transformers Sam2Model: <https://huggingface.co/docs/transformers/main/en/model_doc/sam2>
- ADR 0001 — pretrained-first.
- ADR 0011 — severity-heuristic (dependencia downstream).
- ADR 0012 — mask-encoding-polygon-cap.
