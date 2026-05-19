# ADR 0013 — Tokens CSS auditables + self-host de fuentes

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

El demo se presenta en pantalla compartida durante una entrevista técnica. La paleta y la tipografía deben estar formalizadas como tokens auditables (no hex regados en componentes), y las fuentes deben cargar sin ping a CDN externo (privacidad alineada con `architecture doc` sec 10).

M0 dejó tokens HSL shadcn-style en `frontend/src/styles/globals.css`. M1 añadió hex literales en `frontend/src/lib/colors.ts` para severidad (testeados con Vitest contra valores literales). `frontend/tailwind.config.ts` declara `fontFamily.sans/mono/serif` con Geist Sans / JetBrains Mono / Instrument Serif, pero **no había `@font-face` en ningún lado**: la UI caía a `ui-sans-serif` system fallback silenciosamente.

## Decisión

### Tokens

`globals.css :root` se extiende con:

- **Severidad como CSS vars** (`--severity-leve`, `--severity-moderado`, `--severity-severo` y sus `*-fill` rgba). Los valores son los mismos hex literales que `colors.ts` exporta. Un test `tokens.test.ts` lee `globals.css` como string y assertea coincidencia para evitar drift.
- **Surface tokens** (`--surface-glass`, `--surface-elevated`, `--surface-border`) y un `--radius-lg` (20px) para superficies grandes.
- **Motion tokens** (`--motion-duration-fast/base/slow`, `--motion-ease-out/in-out`) — Framer Motion los consume vía `lib/motion.ts` (hardcoded en TS por simplicidad SSR/test, mantenido en sync manualmente).
- **Tipografía como CSS vars** (`--font-sans/mono/serif`) — el theme dark de ECharts las lee con `getComputedStyle`.

### Fuentes

Tres familias self-hosted en `frontend/public/fonts/`:

- `geist-sans-variable.woff2` (Vercel, OFL — variable 100..900)
- `jetbrains-mono-variable.woff2` (JetBrains, OFL — variable 100..800)
- `instrument-serif-regular.woff2` (Instrument, OFL — 400 normal único weight)

Total ~120 KB, commiteado al repo. nginx ya cachea `woff2|ttf|otf|eot` con `expires 1y immutable` (heredado de M1, ADR 0006).

`@font-face` declarado en `globals.css` con `font-display: swap`. Preload en `index.html` solo de Geist Sans + JetBrains Mono (críticas para UI y datos); Instrument Serif carga lazy con swap.

## Consecuencias

### Positivas

- Paleta auditable: un revisor abre `globals.css` y ve el sistema completo.
- ECharts puede leer la paleta sin importar TS — desacopla el theme dark de React.
- Cero requests a `fonts.googleapis.com` o `fonts.gstatic.com`. Privacy alignment.
- Demo reproducible offline una vez cacheado el HTML.
- Variable woff2 reduce número de archivos (1 archivo vs ~8 weights × 2 styles tradicional).

### Negativas

- ~120 KB binarios commiteados al repo. Aceptable para un demo; se justifica por autonomía CDN.
- Drift potencial entre `globals.css` y `colors.ts` mitigado por `tokens.test.ts` pero requiere mantener ambos en sync.
- `lib/motion.ts` también puede driftar contra `--motion-*` en CSS (sin test anti-drift por limitaciones de jsdom). Mantenido manualmente.

## Alternativas consideradas

1. **Google Fonts CDN.** Rechazada — privacy ping cada visita; latencia ~50–150ms extra primer load.
2. **Build step `scripts/fetch-fonts.sh` con sha256 en postinstall.** Rechazada — añade flakiness al build (depende de GitHub raw URLs); commit binarios es coherente con cómo M3 cachea modelos en volumen.
3. **Tokens solo en `colors.ts` (TS), sin CSS vars.** Rechazada — ECharts no puede importar TS sin acoplar el theme a React; el bundle del theme ya es lo que ECharts pide.

## Referencias

- design doc sec D1+D2.
- [`Docs/decisions/0006-frontend-serving-nginx.md`](0006-frontend-serving-nginx.md) — nginx cache headers para woff2.
- Geist Sans: https://github.com/vercel/geist-font (OFL).
- JetBrains Mono: https://github.com/JetBrains/JetBrainsMono (OFL).
- Instrument Serif: https://github.com/Instrument/instrument-serif (OFL).
