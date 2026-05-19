# ADR 0012 — Mask encoding: polygon simplification con cap de vértices

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno
**Milestone:** M3
**Spec asociado:** design doc sec D3

## Contexto

SAM 2 produce máscara binaria densa H×W. El contrato del API
(`Mask.polygon: list[tuple[float, float]]`, `min_length=3`) y la decisión
M0/M1 sobre overlay SVG (ADR 0005) requieren convertir la máscara a polígono
simplificado.

Sin simplificación, una máscara de 600×400 produce contornos de 200+ puntos
que sobrecargan el SVG y no aportan información visual.

## Decisión

Algoritmo de conversión `mask → polygon` (en `app/pipeline/segmenter.py`):

1. `cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_NONE)` — solo
   contorno externo.
2. Si hay múltiples contornos: tomar el de **mayor área** (`cv2.contourArea`),
   descartar fragmentos espurios.
3. `cv2.approxPolyDP(largest, epsilon=0.005 × diag(bbox), closed=True)`.
4. Si el resultado tiene > 30 vértices: aumentar `epsilon × 1.5` y reintentar
   hasta 5 iteraciones.
5. Si tras 5 iteraciones sigue largo: muestreo equiespaciado por arc-length
   sobre el contorno crudo, quedándose con 30 puntos.
6. Si el polígono tiene < 3 vértices: devolver `None`. El caller
   (`analyze.py`) usa fallback bbox-polygon para mantener el contrato
   `Mask.polygon ≥ 3` (ver ADR 0011).

**Epsilon proporcional a la diagonal del bbox originario, no del polígono crudo.**
Esto mantiene calidad visual estable a través de daños grandes y pequeños.

## Consecuencias

### Positivas

- **SVG performante.** ≤30 vértices × ≤5 daños = ≤150 vértices totales por
  reporte. Muy inferior al overhead aceptable de un SVG.
- **Adaptive epsilon.** Casos patológicos (máscara muy ruidosa, contornos
  fractales) terminan en polígono utilizable; el fallback equispaced
  garantiza terminación.
- **Multi-contour explícito.** Si SAM 2 produce blobs disjuntos por error, se
  conserva el dominante en lugar de unirlos artificialmente.
- **Sin cambios de contrato.** `Mask.polygon` (M0) sigue intacto.

### Negativas

- **Pérdida de detalle en máscaras complejas.** Una grieta en zigzag puede
  perder algunos picos. Mitigación: el demo no requiere pixel-perfect;
  visualmente suficiente para overlay.
- **Multi-contour pierde información.** Si una máscara real tiene dos zonas
  disjuntas legítimas (ej. vidrio roto y luz rota en la misma detección),
  M3 solo conserva una. Aceptable: el detector M2 separa estos casos en
  detecciones distintas.

## Alternativas consideradas

### RLE (Run-Length Encoding)

Rechazado para M3. Cambia el contrato `Mask.polygon` por `Mask.rle: str`,
forzando frontend de SVG a Canvas — gran refactor. M3 reserva RLE como
fallback documentado si en M4 el usuario reporta máscaras visiblemente
pixeladas.

### Cap fijo bajo (≤10 vértices)

Rechazado. Polígonos de 10 vértices no rinden bordes curvos creíbles
(rayones largos, abolladuras circulares). 30 es el sweet spot calidad
visual / overhead SVG.

### Sin simplificación (todos los puntos del contorno)

Rechazado. SVG con 200+ vértices × 5 daños = 1000+ vértices DOM. Latencia
visible en navegadores móviles. Sin ganancia perceptible.

### Múltiples polígonos por máscara

Rechazado. Cambia el contrato `Mask` (un polígono → lista de polígonos).
Cost-benefit no justifica el cambio en M3. Re-evaluable si el caso real
aparece.

## Referencias

- ADR 0005 — overlay-svg-vs-canvas (M1, fija el frontend SVG).
- ADR 0010 — sam2-zero-shot (define el origen de la máscara).
- ADR 0011 — severity-heuristic (consume el polígono).
- OpenCV `approxPolyDP`: <https://docs.opencv.org/4.x/dd/d49/tutorial_py_contour_features.html>
- Algoritmo Douglas–Peucker: <https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm>
