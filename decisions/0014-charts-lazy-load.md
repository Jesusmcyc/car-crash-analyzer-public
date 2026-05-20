# ADR 0014 — ECharts via dynamic import + dark theme with CSS vars

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

The damage report is enriched with two charts: a severity donut and a horizontal bar chart by type. Apache ECharts is the library chosen in `architecture doc` sec 4.2 (mature, dark-friendly, accessible). The ECharts bundle is around 280 KB minified / 85 KB gzipped — importing it statically from `App.tsx` breaks the D7 bundle budget (initial JS <150 KB gzip).

## Decision

### Lazy loading

`SeverityChart` and `DamageTypeChart` are loaded via `React.lazy(() => import(...))`. Both live inside `<Suspense fallback={<ChartSkeleton />}>` in `<ReportPanel />`. Vite produces separate chunks; the first chart to mount pulls echarts+echarts-for-react into the lazy bundle, and the second benefits from the cache.

### Custom dark theme

`buildEchartsTheme()` in `components/report/echarts-theme.ts` builds a theme object by reading the CSS vars of `:root` with `getComputedStyle`. Hardcoded fallbacks for SSR/test. It is registered once per chart (`echarts.registerTheme("cca-dark", buildEchartsTheme())`) and passed as `<ReactECharts theme="cca-dark" />`.

The theme covers:

- A 3-color palette reading `--severity-leve/moderado/severo`.
- `backgroundColor: "transparent"` so the glass surfaces behind it remain visible.
- `textStyle.fontFamily` = `--font-sans` for consistency with the rest of the UI.
- A tooltip with `backdrop-filter: blur(10px)` and a `rgba(11, 18, 32, 0.92)` background (translucent dark navy).
- Soft grid/axis lines (white rgba at 6–20% opacity).

### Accessibility

ECharts does not produce semantic ARIA that is useful for screen readers. M4 wraps each chart in `<div role="img" aria-label="...">` with a label computed from the dataset (e.g. "Severity distribution: 2 minor, 1 moderate, 0 severe").

## Consequences

### Positives

- Clean initial bundle (<150 KB gzip, D7 target) — echarts loads only when there is a report.
- Theme reactive to tokens — a future light/dark toggle is achieved by changing the CSS vars and re-instantiating the chart.
- A semantic `aria-label` is superior to the opaque ECharts SVG for assistive technology.

### Negatives

- 100–300ms of `<ChartSkeleton>` fallback on the first chart (chunk download latency). Acceptable — it happens after the skeleton/success paint and does not affect LCP.
- Two separate chunks even though they share echarts — Vite deduplicates them via a shared chunk, but the split is not perfect. Acceptable; micro-optimization is not pursued.

## Alternatives considered

1. **echarts/core with granular tree-shaking.** Rejected — it increases import complexity (registering series, axes, etc. one by one) and the saving vs full echarts is <30 KB gzip. Not worth it.
2. **Recharts or Chart.js.** Rejected — the project stack already fixes ECharts; Recharts is less performant with animations; Chart.js has worse dark theming.
3. **Hand-built SVG charts.** Rejected — a decent donut with tooltip + legend costs more than the ECharts chunk.

## References

- design doc sec D3.
- ECharts 5: https://echarts.apache.org/handbook/en/get-started.
- React.lazy + Suspense: https://react.dev/reference/react/lazy.
