# ADR 0015 — Motion budget: 3 discrete animations with Framer Motion

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

A demo on a shared screen that overuses animations feels amateur (parallax, bouncy hovers, aggressive springs). A demo with no animations feels static and wireframe-like. The balance is **discrete animations that reinforce information**, not ones that decorate.

## Decision

**Three animations, none of them negotiable:**

| # | Location | Type | Duration | Easing | Trigger |
|---|---|---|---|---|---|
| 1 | Entrance of the `<motion.section>` result | fade + slide-up 12px | 320ms | `cubic-bezier(0.22, 1, 0.36, 1)` | `state.kind` changes from `idle` |
| 2 | Reveal of the overlay polygons | opacity + scale, 80ms stagger | 200ms per polygon | same ease | Mount of `<DamageOverlay>` |
| 3 | Crossfade `SkeletonResult` ↔ success block | opacity + 4px translate-y, `mode="wait"` | 320ms | same ease | `state.kind` `uploading` → `success` |

Any additional animation requires a new ADR. No bouncy hovers, no parallax, no scroll-linked motion, no aggressive spring physics.

`<MotionConfig reducedMotion="user">` wraps the app in `main.tsx` — Framer Motion 11 honors `prefers-reduced-motion: reduce` automatically, disabling the 3 animations when the user requests it at the OS level.

The timing and easing constants live in `frontend/src/lib/motion.ts` (`MOTION.durationBase = 0.32`, `MOTION.easeOut = [0.22, 1, 0.36, 1]`) — the same values exist as the CSS vars `--motion-duration-base` and `--motion-ease-out` in `globals.css`. The duplication is documented in ADR 0013.

## Consequences

### Positives

- Explicit discipline: future temptations to add animations are rejected or require an ADR.
- The stagger in the overlay emphasizes information (each instance of damage appears sequentially — the eye follows one at a time).
- The skeleton↔success crossfade visually reinforces the CLS=0 promise (ADR D5).
- Reduced-motion for free via `<MotionConfig>`.

### Negatives

- `framer-motion` adds ~30 KB gzip to the initial bundle. Acceptable under the D7 target (<150 KB).
- `motion.polygon` (animation 2) uses SVG element animation — supported in Framer Motion 11 but an edge case if regressions appear. Mitigated: the existing `DamageOverlay.test.tsx` tests query native DOM polygons and remain green.

## Alternatives considered

1. **Pure CSS animations (`@keyframes`).** Rejected — without `AnimatePresence` the component's exit does not animate (React unmounts without waiting). Crossfade D4 #3 would not work.
2. **No animations.** Rejected — the demo would be too static for an interview aimed at a senior portfolio.
3. **GSAP or Motion One.** Rejected — Framer Motion is already fixed in the stack; there is no reason to diversify.

## References

- design doc sec D4.
- Framer Motion 11: https://www.framer.com/motion/.
- Material Design Motion: https://m3.material.io/styles/motion (easing reference).
