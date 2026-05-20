# ADR 0005 — DamageOverlay: SVG over <img>, not canvas

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

The damage overlay paints polygons on top of the uploaded image. For M1 (mock with 3 polygons) and M3 (real segmentation with SAM 2) there are two mainstream approaches: an HTML canvas or declarative SVG inside the DOM.

## Decision

Render an absolute-positioned `<svg>` on top of the `<img>`, with a `viewBox` equal to the intrinsic dimensions of the image. Each `Damage` produces a `<polygon>` with a stroke and a semi-transparent fill derived from the severity.

## Consequences

### Positives
- Vitest tests query `<polygon>` directly via Testing Library — no mocks of `getContext("2d")` and no pixel snapshots.
- Hover, damage selection, and tooltips are wired up with idiomatic React event handlers.
- Future zoom and PNG export: SVG scales without loss; a canvas is tied to the device pixel ratio.
- Accessibility: `<svg role="img" aria-label="...">` has better support than canvas.

### Negatives
- With dense masks (hundreds of vertices per polygon), SVG can produce jank when re-rendering. If this shows up in M3, refactor to a canvas or to an isolated (offscreen) `<canvas>`.

## Alternatives considered

1. **HTML5 canvas.** Advantage: performance with thousands of shapes. Rejected for M1–M3 because the expected density (≤5 polygons in M1, ~5 denser ones in M3) is well below the threshold where canvas wins.
2. **WebGL / shaders.** Over-engineering for a demo; rejected.

## References
- design doc sec D2.
