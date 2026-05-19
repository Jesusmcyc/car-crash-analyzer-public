# ADR 0015 — Motion budget: 3 animaciones discretas con Framer Motion

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

Un demo en pantalla compartida que abusa de animaciones se siente amateur (parallax, hovers bouncy, springs agresivos). Un demo sin animaciones se siente estático y wireframe. El balance es **animaciones discretas que refuerzan la información**, no que decoran.

## Decisión

**Tres animaciones, ninguna negociable:**

| # | Ubicación | Tipo | Duración | Easing | Trigger |
|---|---|---|---|---|---|
| 1 | Entrada de `<motion.section>` resultado | fade + slide-up 12px | 320ms | `cubic-bezier(0.22, 1, 0.36, 1)` | `state.kind` cambia de `idle` |
| 2 | Reveal de polígonos del overlay | opacity + scale, stagger 80ms | 200ms por polígono | mismo ease | Mount de `<DamageOverlay>` |
| 3 | Crossfade `SkeletonResult` ↔ success block | opacity + 4px translate-y, `mode="wait"` | 320ms | mismo ease | `state.kind` `uploading` → `success` |

Toda animación adicional requiere ADR nuevo. Sin hover bouncy, sin parallax, sin scroll-linked, sin spring physics agresivos.

`<MotionConfig reducedMotion="user">` envuelve la app en `main.tsx` — Framer Motion 11 honra `prefers-reduced-motion: reduce` automáticamente, deshabilitando las 3 animaciones cuando el usuario lo pide a nivel SO.

Constantes de tiempo y easing viven en `frontend/src/lib/motion.ts` (`MOTION.durationBase = 0.32`, `MOTION.easeOut = [0.22, 1, 0.36, 1]`) — los mismos valores existen como CSS vars `--motion-duration-base` y `--motion-ease-out` en `globals.css`. La duplicación está documentada en ADR 0013.

## Consecuencias

### Positivas

- Disciplina explícita: futuras tentaciones de añadir animaciones se rechazan o requieren ADR.
- Stagger en el overlay subraya información (cada daño aparece secuencialmente — el ojo sigue uno a la vez).
- Crossfade skeleton↔success refuerza visualmente la promesa de CLS=0 (ADR D5).
- Reduced-motion gratis vía `<MotionConfig>`.

### Negativas

- `framer-motion` añade ~30 KB gzip al initial bundle. Aceptable bajo el target D7 (<150 KB).
- `motion.polygon` (animación 2) usa SVG element animation — soportado en Framer Motion 11 pero un edge si hay regresiones. Mitigado: existing tests `DamageOverlay.test.tsx` queryan polygons DOM nativo y siguen verdes.

## Alternativas consideradas

1. **Animaciones CSS puras (`@keyframes`).** Rechazada — sin `AnimatePresence` el exit del componente no anima (React desmonta sin esperar). Crossfade D4 #3 no funcionaría.
2. **Sin animaciones.** Rechazada — el demo queda demasiado estático para una entrevista que apunta a portfolio senior.
3. **GSAP o Motion One.** Rechazadas — Framer Motion ya está fijo en el stack; no hay razón para diversificar.

## Referencias

- design doc sec D4.
- Framer Motion 11: https://www.framer.com/motion/.
- Material Design Motion: https://m3.material.io/styles/motion (referencia de easings).
