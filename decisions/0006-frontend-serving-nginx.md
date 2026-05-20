# ADR 0006 — Frontend in production: nginx-alpine multi-stage, not `vite preview`

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

The Vite frontend generates static assets. They have to be served from the container that ships to Coolify. Options considered: `vite preview` (Vite's dev server), `serve`/`http-server` (Node), nginx-alpine, Caddy.

## Decision

Multi-stage Docker:
- A `node:20-alpine` builder runs `npm ci && npm run build`.
- A `nginx:alpine` runtime with its own `nginx.conf` — gzip, SPA fallback to `index.html`, cache headers (`immutable` for hashed assets, `no-cache` for `index.html`).

Final image ~25 MB.

## Consequences

### Positives
- Industry standard for serving static SPAs.
- Explicit cache + gzip configuration — critical for a public demo.
- Minimal runtime image; no Node or `node_modules` in production.
- nginx handles concurrent connections orders of magnitude better than a casual Node server.

### Negatives
- One more piece of configuration (`nginx.conf`) to maintain.
- Changes to SPA fallback rules require rebuilding the image.

## Alternatives considered

1. **`vite preview`.** Intended only to verify the build, not for production; the Vite documentation says so. Rejected.
2. **Caddy.** Automatic TLS is attractive, but Coolify already terminates TLS; Caddy would duplicate responsibilities. Rejected.
3. **`serve` (Node).** It works, but it drags Node and `node_modules` into the runtime image (~120MB+ vs ~25MB). Rejected.

## References
- design doc sec D3.
