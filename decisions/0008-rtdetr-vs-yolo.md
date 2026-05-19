# ADR 0008 — Detector default: RT-DETRv2 sobre YOLOv8

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

M2 introduce el primer modelo real al pipeline reemplazando el mock determinista de M1. Dos candidatos pretrained vía Ultralytics: RT-DETRv2 (transformer DETR-style) y YOLOv8 (CNN one-stage). Ambos resuelven detección general; ambos vienen entrenados sobre COCO; ambos exponen API idéntica en Ultralytics.

La pregunta no es "cuál es mejor en abstracto" — es **cuál sirve mejor al caso de uso del demo (CarDD, ~4k imágenes, daños solapados, CPU)**.

## Decisión

**Default: RT-DETRv2** (peso `rtdetr-l.pt`). **Fallback funcional: YOLOv8** (peso `yolov8s.pt`). Switch por env var `DETECTOR_BACKEND ∈ {"rtdetr", "yolov8"}` sin recompilar.

Mapeo COCO → DamageType: **placeholder explícito hasta M5**. Pretrained COCO no conoce `dent`/`scratch`/etc., así que M2 detecta vehículos (clases `car`, `truck`, `bus`, `motorcycle`) y les asigna un DamageType cíclico. M5 fine-tunea sobre CarDD y los pesos custom devuelven clases reales del enum.

## Consecuencias

### Positivas

- **Set prediction sin NMS.** Daños vehiculares se solapan en CarDD (rayón sobre abolladura, vidrio roto adyacente a marco doblado). RT-DETR no colapsa detecciones cercanas como YOLOv8 con NMS tiende a hacer. Ventaja que se materializa cuando los pesos custom de M5 entren al pipeline.
- **Sample efficiency.** CarDD tiene ~4k imágenes — pequeño. Los DETRs aprenden mejor en datasets de este tamaño que YOLO from scratch.
- **API unificada Ultralytics.** El costo de mantener YOLOv8 como fallback es prácticamente cero — misma firma `Model(image)` → `Result.boxes`. La factory `get_detector(settings)` selecciona y todo el pipeline lo consume opaco.
- **Honestidad técnica.** El placeholder COCO→DamageType queda documentado prominentemente (este ADR + el módulo). El demo no miente: M2 valida el plumbing del pipeline; M5 lo califica como detector real de daños.

### Negativas

- **Detecciones M2 no son específicas de daños.** El demo en producción tras M2 muestra "este pipeline detecta vehículos en imágenes" más que "este pipeline detecta daños vehiculares". Mitigado por M5 (paralelo) que reemplaza pesos por env var sin tocar código.
- **RT-DETR-l es ~70 MB vs YOLOv8s ~22 MB.** Cold-start del primer boot del VPS es 3× más largo si arranca con RT-DETR. Mitigado por cache en volumen (ADR 0009).
- **Latencia inferencia CPU mayor en RT-DETR.** Esperado 5–8s vs 2–3s YOLOv8. Aceptable para demo (no SLO de producción); fallback YOLOv8 disponible si M5 muestra que el upgrade no compensa la latencia.

## Alternativas consideradas

1. **YOLOv8 como default.** Más rápido en CPU, descarga más liviana. Rechazada porque el problema (daños solapados) es donde RT-DETR brilla — fijar el default basándose solo en latencia sacrificaría la ventaja arquitectónica.
2. **Mantener mock hasta M5.** Sin pipeline real, el deploy de M1 nunca valida que el container Docker aguanta PyTorch + Ultralytics + cache de pesos. Riesgo de descubrir el problema al final del proyecto. Rechazada.
3. **Pesos pretrained sobre dataset más cercano (Open Images, no COCO).** Open Images tampoco tiene clases de daño vehicular. Mismo placeholder problem; sin upside. Rechazada.

## Referencias

- Spec maestro: design doc sec M2 y sec 4 (ADRs).
- Design decisions M2: design doc sec 2 D2 + D3.
- ADR 0001 — pretrained-first: [`0001-pretrained-first.md`](0001-pretrained-first.md).
- Implementación: `backend/app/pipeline/detector.py`.
