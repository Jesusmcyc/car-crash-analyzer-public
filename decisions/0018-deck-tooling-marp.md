# ADR 0018 — Deck tooling: Marp

**Status:** Accepted
**Date:** 2026-05-06
**Author:** Jesús Moreno
**Milestone:** M6

## Context

Closing out the project requires a slide deck for a technical interview at Momento Seguros (screen-shared, ≤10 min). Three real options: Marp (markdown → PDF), Slidev (Vue/Vite with a dev server), Google Slides (cloud, opaque binary).

## Decision

**Marp.** Source in `docs/deck/slides.md`, exported to `docs/deck/slides.pdf` with `npx @marp-team/marp-cli@latest --allow-local-files --pdf docs/deck/slides.md -o docs/deck/slides.pdf`. The static PDF is committed so as not to depend on a toolchain during the interview.

## Consequences

### Positives

- The deck is committed as markdown — line-level diffs, per-commit blame, review in PRs, aligned with the rest of the project (ADRs, specs, and metrics are all markdown).
- A static PDF does not break Meet/Zoom screen sharing (the decisive factor for a screen-shared interview).
- Zero runtime dependency during the presentation; the system viewer is enough.
- `--allow-local-files` makes it possible to embed `assets/demo.gif` and `assets/diagrams/*.png` without serving a dev server.

### Negatives

- Presenter mode is more limited than Slidev (no rich transitions, no live editor during the session). Mitigation: the notes live in the speaker's head after the dry runs (D6).
- Requires node 20 + npm for the export. Mitigation: this is already covered by the project's frontend; an additional alternative is `docker run marpteam/marp-cli`.

### Neutral

- Marp supports `<!-- _class: lead -->` for cover/closing slides and `---` as the delimiter between slides; standard markdown syntax with no mandatory extensions.

## Alternatives considered

1. **Slidev** — better presenter mode and animations, but requires a live dev server during the presentation. Rejected because of the failure surface in screen sharing.
2. **Google Slides** — an opaque binary with a revision history but no auditable technical diff. Requires network + login at the critical moment. Rejected.
3. **PowerPoint / Keynote** — proprietary formats; require the app to be installed; fonts may not embed. Rejected for portability.
4. **Manual Reveal.js** — more control but more friction to maintain; Marp already wraps Reveal-style syntax without requiring manual HTML.
5. **LaTeX + Beamer** — overkill for 10 portfolio slides. The sweet spot is markdown.

## References

- Spec M6 D1: design doc
- Marp CLI: <https://github.com/marp-team/marp-cli>.
- Marp documentation: <https://marp.app>.
