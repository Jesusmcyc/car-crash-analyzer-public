# ADR 0025 — Motion budget: expand to 7 with flow vs scroll categories

**Status:** Accepted
**Date:** 2026-05-18
**Author:** Jesús Moreno
**Milestone:** M8 (frontend redesign)
**Supersedes:** ADR 0015

## Context

ADR 0015 fixed the motion budget at exactly 3 Framer Motion animations, justified for a single page without narrative scroll. The M8 redesign adds five new sections (CoverHero, PipelineSection, BenchmarksSection, AdrsSection, StackSection) whose visual language requires viewport-triggered stagger to feel alive without gratuitous animation. The rigid budget of 3 does not cover this case.

## Decision

Replace the single counter with two categories with separate counts:

- **Flow (max 4)** — transitions of the app's state machine (`idle → uploading → success → error`) or the initial mount of the hero. Triggered by state, not by scroll. The 3 originals from ADR 0015 + the CoverHero mount = 4.

- **Scroll/section (max 3)** — entrance of long sections, viewport-triggered with `once: true` and `viewport.margin: "-120px"`. Currently: Pipeline stagger, Benchmarks stagger, ADRs fade. A single activation per section on the first scroll.

**Total: 7 Framer Motion animations.** Any new animation must fit into a category or requires an ADR.

What does NOT count toward the budget:
- CSS transitions (hover, focus, color shifts)
- CSS-only decorative animations (`AppBackground` blobs)
- Skeleton shimmer (Tailwind `animate-pulse`)
- Navbar background transition on scroll (CSS transition)

## Consequences

### Positives
- Allows tasteful stagger in new sections without breaking discipline
- The two categories force a classification before animating
- Keeps the rule: zero gratuitous animation

### Negatives
- More complexity to audit (it used to be trivial: count `motion.*` in the repo)
- The category decision can be ambiguous in edge cases

## Alternatives considered

- A qualitative rule "state animations only": simpler but less auditable. Discarded — the count is the discipline.
- Keep 3 and use CSS `animation-timeline: view()` for scroll: inconsistent browser support (Safari < 17.4). Discarded.
- Ignore ADR 0015 without a replacement: incoherent with the project's ADR practice.

## References

- ADR 0015 (the original motion budget of 3) — superseded
- Redesign spec: design doc
