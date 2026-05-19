# ADR 0018 — Deck tooling: Marp

**Estado:** aceptado
**Fecha:** 2026-05-06
**Autor:** Jesús Moreno
**Milestone:** M6

## Contexto

El cierre del proyecto requiere un slide deck para una entrevista técnica en Momento Seguros (pantalla compartida, ≤10 min). Tres opciones reales: Marp (markdown → PDF), Slidev (Vue/Vite con dev server), Google Slides (cloud, binario opaco).

## Decisión

**Marp.** Source en `docs/deck/slides.md`, export a `docs/deck/slides.pdf` con `npx @marp-team/marp-cli@latest --allow-local-files --pdf docs/deck/slides.md -o docs/deck/slides.pdf`. PDF estático commiteado para no depender de toolchain durante la entrevista.

## Consecuencias

### Positivas

- Deck commiteado en markdown — diff por línea, blame por commit, review en PR, alineado con el resto del proyecto (ADRs, specs, métricas son markdown).
- PDF estático no rompe screen share de Meet/Zoom (factor decisivo para entrevista en pantalla compartida).
- Cero dependencia de runtime durante la presentación; el visor del sistema basta.
- `--allow-local-files` permite embeber `assets/demo.gif` y `assets/diagrams/*.png` sin servir un dev server.

### Negativas

- Presenter mode más limitado que Slidev (no rich transitions, no live editor durante la sesión). Mitigación: las notas viven en la cabeza del speaker tras los dry-runs (D6).
- Requiere node 20 + npm para el export. Mitigación: ya está cubierto por el frontend del proyecto; alternativa adicional con `docker run marpteam/marp-cli`.

### Neutrales

- Marp soporta `<!-- _class: lead -->` para slides de portada/cierre y `---` como delimitador entre slides; sintaxis estándar markdown sin extensiones obligatorias.

## Alternativas consideradas

1. **Slidev** — mejor presenter mode y animaciones, pero requiere dev server vivo durante la presentación. Rechazado por superficie de fallo en pantalla compartida.
2. **Google Slides** — binario opaco con revision history sin diff técnico auditable. Requiere red + login en el momento crítico. Rechazado.
3. **PowerPoint / Keynote** — formatos propietarios; requieren la app instalada; fuentes pueden no embarcar. Rechazado por portabilidad.
4. **Reveal.js manual** — más control pero más fricción para mantener; Marp ya wrappear sintaxis Reveal-style sin requerir HTML manual.
5. **LaTeX + Beamer** — overkill para 10 slides de portfolio. El sweet spot es markdown.

## Referencias

- Spec M6 D1: design doc
- Marp CLI: <https://github.com/marp-team/marp-cli>.
- Marp documentation: <https://marp.app>.
