# ADR 0025 — Motion budget: ampliar a 7 con categorías flujo vs scroll

**Estado:** Aceptado
**Fecha:** 2026-05-18
**Autor:** Jesús Moreno
**Milestone:** M8 (rediseño del frontend)
**Supersedes:** ADR 0015

## Contexto

ADR 0015 fijó el motion budget en exactamente 3 animaciones Framer Motion, justificado para single-page sin scroll narrativo. El rediseño M8 agrega cinco secciones nuevas (CoverHero, PipelineSection, BenchmarksSection, AdrsSection, StackSection) cuyo lenguaje visual requiere stagger viewport-triggered para sentirse vivas sin animación gratuita. El budget rígido de 3 no cubre este caso.

## Decisión

Reemplazar el contador único por dos categorías con conteo separado:

- **Flujo (máx 4)** — transiciones del state machine de la app (`idle → uploading → success → error`) o mount inicial del hero. Disparadas por estado, no por scroll. Las 3 originales de ADR 0015 + el mount del CoverHero = 4.

- **Scroll/sección (máx 3)** — entrada de secciones largas, viewport-triggered con `once: true` y `viewport.margin: "-120px"`. Hoy: Pipeline stagger, Benchmarks stagger, ADRs fade. Una sola activación por sección al primer scroll.

**Total: 7 animaciones Framer Motion.** Cualquier animación nueva debe encajar en una categoría o requiere ADR.

Lo que NO cuenta hacia el budget:
- CSS transitions (hover, focus, color shifts)
- Animaciones decorativas CSS-only (`AppBackground` blobs)
- Skeleton shimmer (Tailwind `animate-pulse`)
- Navbar transición de fondo al scrollear (CSS transition)

## Consecuencias

### Positivas
- Permite stagger educado en secciones nuevas sin romper disciplina
- Las dos categorías obligan a clasificar antes de animar
- Mantiene la regla: cero animación gratuita

### Negativas
- Más complejidad para auditar (antes era trivial: contar `motion.*` en el repo)
- Decisión de categoría puede ser ambigua en casos edge

## Alternativas consideradas

- Regla cualitativa "solo animaciones de estado": más simple pero menos auditable. Descartada — el conteo es la disciplina.
- Mantener 3 y usar `animation-timeline: view()` CSS para scroll: soporte de browser inconsistente (Safari < 17.4). Descartada.
- Ignorar ADR 0015 sin reemplazo: incoherente con la práctica de ADRs del proyecto.

## Referencias

- ADR 0015 (motion budget original de 3) — superseded
- Spec del rediseño: design doc
