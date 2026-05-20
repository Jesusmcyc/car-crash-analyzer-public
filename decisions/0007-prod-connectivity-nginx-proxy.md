# ADR 0007 — Production connectivity: same-origin via nginx `/api` proxy

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

In the Coolify deployment (Traefik terminates TLS and routes by domain to a single service), the frontend (a static SPA served by `nginx:alpine`) and the backend (uvicorn) run as separate containers on the same Docker network, but the end client's browser is outside that network.

The frontend's HTTP client (`frontend/src/api/client.ts`) reads `VITE_API_BASE_URL` at build time, with a default of `http://localhost:8000`. If the prod image ships with that default, the browser tries to talk to `http://localhost:8000` (the user's own machine), not the real backend, and the `/analyze` endpoint fails.

Three patterns to solve it:

1. Same domain + reverse proxy in the frontend nginx (`/api/*` → `backend:8000/*`).
2. Two separate domains (`demo.com` for the frontend, `api.demo.com` for the backend) with CORS.
3. Path-based routing in Coolify's reverse proxy (Traefik labels) directly to two services.

## Decision

Pattern 1: same-origin with `/api` proxied by the frontend's nginx.

Concrete changes:

- `frontend/nginx.conf` adds a `location /api/ { proxy_pass http://backend:8000/; }` with standard proxy headers (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Request-ID`) and `proxy_read_timeout 60s`. `client_max_body_size 12m` leaves headroom over `MAX_UPLOAD_BYTES` (10 MB).
- `frontend/Dockerfile` declares `ARG VITE_API_BASE_URL=/api` and exposes it as an `ENV` before `npm run build`. The prod image ends up with `/api` baked in.
- `docker-compose.prod.yml` passes `args: { VITE_API_BASE_URL: "/api" }` to the frontend build (explicit; it does not rely on the Dockerfile default).
- Local dev (`docker-compose.yml` and `vite dev`) is left untouched: the browser is on `localhost`, and the client's `http://localhost:8000` default works because the dev backend exposes `:8000` to the host.
- `CORS_ORIGINS` in prod ends up effectively unused (same origin, no preflight); it is kept as optional for future scenarios.

## Consequences

### Positives
- A single public domain — it fits Coolify's "one service per domain" model without custom labels.
- No CORS, no preflight `OPTIONS` — one fewer round-trip per request.
- HTTPS terminates at Coolify and the internal `backend:8000` piece is never exposed to the internet.
- `X-Request-ID` crosses the proxy and the backend middleware honors it — the request_id trace is preserved end to end.
- Portable: if we swap Coolify tomorrow for another PaaS or for k8s, the nginx pattern keeps working with no changes to the app.

### Negatives
- One extra nginx hop on every request (~1 ms on a LAN; negligible).
- `proxy_read_timeout 60s` covers the mock; in M3, when real inference can take longer, it has to be raised and/or streaming has to be introduced. A known risk with a trivial mitigation.
- `/api/*` is reserved in the frontend — an asset accidentally placed under `/api/...` would collide with the proxy. There are no such assets today and the convention is documented.
- `VITE_API_BASE_URL` is baked into the image — for additional environments (staging) a rebuild is required. Acceptable: there is only one demo.

## Alternatives considered

1. **Two domains + CORS.** Duplicates the TLS certificate, requires `CORS_ORIGINS` to be configured, and adds DNS propagation. Operationally more expensive for a single-team demo. Rejected.
2. **Backend exposed publicly with open CORS.** Increases the attack surface (rate-limiting and auth would have to harden a directly exposed `/analyze`). Rejected for a public portfolio.
3. **Path-based routing with Traefik labels in `docker-compose.prod.yml`.** It works, but it ties the repo to Coolify/Traefik. Rejected for portability.

## References

- design doc sec D3 (frontend serving in nginx).
- [`Docs/decisions/0006-frontend-serving-nginx.md`](0006-frontend-serving-nginx.md) — the foundation this ADR extends.
- `backend/app/api/middleware.py` — `RequestIdMiddleware`, which consumes the `X-Request-ID` propagated by the proxy.
