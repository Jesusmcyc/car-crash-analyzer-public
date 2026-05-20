# ADR 0012 — Mask encoding: polygon simplification with vertex cap

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno
**Milestone:** M3
**Related spec:** design doc sec D3

## Context

SAM 2 produces a dense H×W binary mask. The API contract
(`Mask.polygon: list[tuple[float, float]]`, `min_length=3`) and the
M0/M1 decision on the SVG overlay (ADR 0005) require converting the mask
to a simplified polygon.

Without simplification, a 600×400 mask produces contours of 200+ points
that overload the SVG and add no visual information.

## Decision

`mask → polygon` conversion algorithm (in `app/pipeline/segmenter.py`):

1. `cv2.findContours(mask, cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_NONE)` — outer
   contour only.
2. If there are multiple contours: take the one with the **largest area**
   (`cv2.contourArea`), discarding spurious fragments.
3. `cv2.approxPolyDP(largest, epsilon=0.005 × diag(bbox), closed=True)`.
4. If the result has > 30 vertices: increase `epsilon × 1.5` and retry
   for up to 5 iterations.
5. If it is still too long after 5 iterations: equispaced arc-length
   sampling over the raw contour, keeping 30 points.
6. If the polygon has < 3 vertices: return `None`. The caller
   (`analyze.py`) uses a bbox-polygon fallback to keep the
   `Mask.polygon ≥ 3` contract (see ADR 0011).

**Epsilon is proportional to the diagonal of the original bbox, not of the raw polygon.**
This keeps visual quality stable across large and small damage.

## Consequences

### Positives

- **Performant SVG.** ≤30 vertices × ≤5 instances of damage = ≤150 total
  vertices per report. Well below the acceptable overhead of an SVG.
- **Adaptive epsilon.** Pathological cases (very noisy mask, fractal
  contours) end up as a usable polygon; the equispaced fallback
  guarantees termination.
- **Explicit multi-contour handling.** If SAM 2 erroneously produces
  disjoint blobs, the dominant one is kept instead of artificially
  merging them.
- **No contract changes.** `Mask.polygon` (M0) remains intact.

### Negatives

- **Loss of detail on complex masks.** A zigzag crack may lose some
  peaks. Mitigation: the demo does not require pixel-perfect output;
  it is visually sufficient for an overlay.
- **Multi-contour loses information.** If a real mask has two legitimate
  disjoint zones (e.g. broken glass and a broken light in the same
  detection), M3 keeps only one. Acceptable: the M2 detector separates
  these cases into distinct detections.

## Alternatives considered

### RLE (Run-Length Encoding)

Rejected for M3. It replaces the `Mask.polygon` contract with
`Mask.rle: str`, forcing the frontend from SVG to Canvas — a large
refactor. M3 keeps RLE as a documented fallback in case the user
reports visibly pixelated masks in M4.

### Low fixed cap (≤10 vertices)

Rejected. Polygons of 10 vertices do not render believable curved edges
(long scratches, circular dents). 30 is the sweet spot between visual
quality and SVG overhead.

### No simplification (all contour points)

Rejected. An SVG with 200+ vertices × 5 instances of damage = 1000+ DOM
vertices. Visible latency on mobile browsers. No perceptible gain.

### Multiple polygons per mask

Rejected. It changes the `Mask` contract (one polygon → list of
polygons). The cost-benefit does not justify the change in M3.
Re-evaluable if the real-world case appears.

## References

- ADR 0005 — overlay-svg-vs-canvas (M1, fixes the frontend on SVG).
- ADR 0010 — sam2-zero-shot (defines the origin of the mask).
- ADR 0011 — severity-heuristic (consumes the polygon).
- OpenCV `approxPolyDP`: <https://docs.opencv.org/4.x/dd/d49/tutorial_py_contour_features.html>
- Douglas–Peucker algorithm: <https://en.wikipedia.org/wiki/Ramer%E2%80%93Douglas%E2%80%93Peucker_algorithm>
