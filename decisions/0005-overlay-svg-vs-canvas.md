# ADR 0005 — DamageOverlay: SVG sobre <img>, no canvas

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

El overlay de daños pinta polígonos sobre la imagen subida. Para M1 (mock con 3 polígonos) y M3 (segmentación real con SAM 2) hay dos enfoques mainstream: HTML canvas o SVG declarativo dentro del DOM.

## Decisión

Renderizar un `<svg>` absolute-positioned encima de la `<img>` con `viewBox` igual a las dimensiones intrínsecas de la imagen. Cada `Damage` produce un `<polygon>` con stroke y fill semitransparente derivados de la severidad.

## Consecuencias

### Positivas
- Tests Vitest queryan `<polygon>` directo via Testing Library — sin mocks de `getContext("2d")` ni snapshots de píxeles.
- Hover, selección de daño y tooltips se montan con event handlers React idiomáticos.
- Zoom y export PNG futuros: SVG escala sin pérdida; canvas queda atado al device pixel ratio.
- Accesibilidad: `<svg role="img" aria-label="...">` tiene mejor soporte que canvas.

### Negativas
- Con máscaras densas (cientos de vértices por polígono) SVG puede generar jank al re-renderizar. Si se manifiesta en M3, refactorizar a canvas o a `<canvas>` aislado (offscreen).

## Alternativas consideradas

1. **Canvas HTML5.** Ventaja: rendimiento en miles de shapes. Rechazada para M1–M3 porque la densidad esperada (≤5 polígonos en M1, ~5 más densos en M3) está muy por debajo del umbral donde canvas gana.
2. **WebGL / shaders.** Sobre-ingeniería para un demo; rechazada.

## Referencias
- design doc sec D2.
