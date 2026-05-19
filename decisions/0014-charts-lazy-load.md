# ADR 0014 — ECharts vía dynamic import + theme dark con CSS vars

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

El reporte de daños se enriquece con dos charts: un donut de severidad y un bar horizontal por tipo. Apache ECharts es la librería elegida en `architecture doc` sec 4.2 (madura, dark-friendly, accesible). El bundle de ECharts ronda 280 KB minified / 85 KB gzip — importarlo estáticamente desde `App.tsx` rompe el bundle budget D7 (initial JS <150 KB gzip).

## Decisión

### Lazy loading

`SeverityChart` y `DamageTypeChart` se cargan via `React.lazy(() => import(...))`. Ambos viven dentro de `<Suspense fallback={<ChartSkeleton />}>` en `<ReportPanel />`. Vite produce chunks separados; el primer chart en montar trae echarts+echarts-for-react al bundle lazy, el segundo se beneficia del cache.

### Theme dark custom

`buildEchartsTheme()` en `components/report/echarts-theme.ts` construye un objeto theme leyendo CSS vars de `:root` con `getComputedStyle`. Fallbacks hardcoded para SSR/test. Se registra una vez por chart (`echarts.registerTheme("cca-dark", buildEchartsTheme())`) y se pasa como `<ReactECharts theme="cca-dark" />`.

Theme contempla:

- Paleta de 3 colores leyendo `--severity-leve/moderado/severo`.
- `backgroundColor: "transparent"` para que las superficies glass de fondo se vean.
- `textStyle.fontFamily` = `--font-sans` para coherencia con el resto de la UI.
- Tooltip con `backdrop-filter: blur(10px)` y bg `rgba(11, 18, 32, 0.92)` (dark navy translúcido).
- Grid/axis lines suaves (rgba blancos con 6–20% opacity).

### Accesibilidad

ECharts no produce ARIA semántico que sirva para screen readers. M4 envuelve cada chart en `<div role="img" aria-label="...">` con label computado del dataset (ej. "Distribución de severidad: 2 leve, 1 moderado, 0 severo").

## Consecuencias

### Positivas

- Initial bundle limpio (<150 KB gzip target D7) — echarts solo carga cuando hay reporte.
- Theme reactivo a tokens — futuro toggle light/dark se logra cambiando CSS vars y re-instanciando el chart.
- `aria-label` semántico supera al SVG opaco de ECharts para tecnología asistiva.

### Negativas

- 100–300ms de fallback `<ChartSkeleton>` en el primer chart (latencia del chunk download). Aceptable — sucede después del paint de skeleton/success y no afecta LCP.
- Dos chunks separados aunque comparten echarts — Vite los deduplica via shared chunk pero el split no es perfecto. Aceptable, no se persigue micro-optimización.

## Alternativas consideradas

1. **echarts/core con tree-shaking granular.** Rechazada — incrementa la complejidad de imports (registrar series, axes, etc. uno por uno) y el ahorro vs full echarts es <30 KB gzip. No vale.
2. **Recharts o Chart.js.** Rechazadas — el stack del proyecto ya fija ECharts; Recharts es menos performante con animaciones; Chart.js tiene dark theming peor.
3. **Charts SVG hechos a mano.** Rechazada — un donut decente con tooltip + legend cuesta más que el chunk de ECharts.

## Referencias

- design doc sec D3.
- ECharts 5: https://echarts.apache.org/handbook/en/get-started.
- React.lazy + Suspense: https://react.dev/reference/react/lazy.
