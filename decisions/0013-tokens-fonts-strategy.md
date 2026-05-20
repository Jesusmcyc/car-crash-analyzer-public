# ADR 0013 — Auditable CSS tokens + self-hosted fonts

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

The demo is presented on a shared screen during a technical interview. The palette and typography must be formalized as auditable tokens (not hex values scattered across components), and fonts must load without pinging an external CDN (privacy aligned with `architecture doc` sec 10).

M0 left shadcn-style HSL tokens in `frontend/src/styles/globals.css`. M1 added literal hex values in `frontend/src/lib/colors.ts` for severity (tested with Vitest against literal values). `frontend/tailwind.config.ts` declares `fontFamily.sans/mono/serif` with Geist Sans / JetBrains Mono / Instrument Serif, but **there was no `@font-face` anywhere**: the UI silently fell back to the `ui-sans-serif` system font.

## Decision

### Tokens

`globals.css :root` is extended with:

- **Severity as CSS vars** (`--severity-leve`, `--severity-moderado`, `--severity-severo` and their `*-fill` rgba values). The values are the same literal hex values that `colors.ts` exports. A `tokens.test.ts` test reads `globals.css` as a string and asserts a match to prevent drift.
- **Surface tokens** (`--surface-glass`, `--surface-elevated`, `--surface-border`) and a `--radius-lg` (20px) for large surfaces.
- **Motion tokens** (`--motion-duration-fast/base/slow`, `--motion-ease-out/in-out`) — Framer Motion consumes them via `lib/motion.ts` (hardcoded in TS for SSR/test simplicity, kept in sync manually).
- **Typography as CSS vars** (`--font-sans/mono/serif`) — the ECharts dark theme reads them with `getComputedStyle`.

### Fonts

Three self-hosted families in `frontend/public/fonts/`:

- `geist-sans-variable.woff2` (Vercel, OFL — variable 100..900)
- `jetbrains-mono-variable.woff2` (JetBrains, OFL — variable 100..800)
- `instrument-serif-regular.woff2` (Instrument, OFL — single 400 normal weight)

Total ~120 KB, committed to the repo. nginx already caches `woff2|ttf|otf|eot` with `expires 1y immutable` (inherited from M1, ADR 0006).

`@font-face` is declared in `globals.css` with `font-display: swap`. Preload in `index.html` covers only Geist Sans + JetBrains Mono (critical for UI and data); Instrument Serif loads lazily with swap.

## Consequences

### Positives

- Auditable palette: a reviewer opens `globals.css` and sees the complete system.
- ECharts can read the palette without importing TS — it decouples the dark theme from React.
- Zero requests to `fonts.googleapis.com` or `fonts.gstatic.com`. Privacy alignment.
- Demo reproducible offline once the HTML is cached.
- The variable woff2 reduces the number of files (1 file vs ~8 weights × 2 styles in the traditional approach).

### Negatives

- ~120 KB of binaries committed to the repo. Acceptable for a demo; justified by CDN autonomy.
- Potential drift between `globals.css` and `colors.ts`, mitigated by `tokens.test.ts` but requiring both to be kept in sync.
- `lib/motion.ts` can also drift against the `--motion-*` CSS vars (no anti-drift test due to jsdom limitations). Maintained manually.

## Alternatives considered

1. **Google Fonts CDN.** Rejected — privacy ping on every visit; ~50–150ms of extra latency on first load.
2. **Build step `scripts/fetch-fonts.sh` with sha256 in postinstall.** Rejected — adds flakiness to the build (depends on GitHub raw URLs); committing binaries is consistent with how M3 caches models in a volume.
3. **Tokens only in `colors.ts` (TS), no CSS vars.** Rejected — ECharts cannot import TS without coupling the theme to React; the theme bundle is exactly what ECharts needs.

## References

- design doc sec D1+D2.
- [`Docs/decisions/0006-frontend-serving-nginx.md`](0006-frontend-serving-nginx.md) — nginx cache headers for woff2.
- Geist Sans: https://github.com/vercel/geist-font (OFL).
- JetBrains Mono: https://github.com/JetBrains/JetBrainsMono (OFL).
- Instrument Serif: https://github.com/Instrument/instrument-serif (OFL).
