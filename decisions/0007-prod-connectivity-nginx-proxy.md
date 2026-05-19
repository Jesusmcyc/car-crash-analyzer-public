# ADR 0007 — Conectividad prod: same-origin via nginx `/api` proxy

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

En el deploy a Coolify (Traefik termina TLS y rutea por dominio a un único servicio), el frontend (SPA estático servido por `nginx:alpine`) y el backend (uvicorn) corren como contenedores separados sobre la misma red de Docker, pero el navegador del cliente final está fuera de esa red.

El cliente HTTP del frontend (`frontend/src/api/client.ts`) lee `VITE_API_BASE_URL` en build-time, con default `http://localhost:8000`. Si la imagen prod sale con ese default, el browser intenta hablarle a `http://localhost:8000` (la máquina del usuario), no al backend real, y el endpoint `/analyze` falla.

Tres patrones para resolverlo:

1. Mismo dominio + reverse proxy en el frontend nginx (`/api/*` → `backend:8000/*`).
2. Dos dominios separados (`demo.com` para frontend, `api.demo.com` para backend) con CORS.
3. Routing path-based en el reverse proxy de Coolify (Traefik labels) directo a dos servicios.

## Decisión

Patrón 1: same-origin con `/api` proxeado por el nginx del frontend.

Cambios concretos:

- `frontend/nginx.conf` añade un `location /api/ { proxy_pass http://backend:8000/; }` con headers de proxy estándar (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Request-ID`) y `proxy_read_timeout 60s`. `client_max_body_size 12m` deja margen sobre `MAX_UPLOAD_BYTES` (10 MB).
- `frontend/Dockerfile` declara `ARG VITE_API_BASE_URL=/api` y lo expone como `ENV` antes de `npm run build`. La imagen prod queda con `/api` baked-in.
- `docker-compose.prod.yml` pasa `args: { VITE_API_BASE_URL: "/api" }` al build del frontend (explícito; no depende del default del Dockerfile).
- Dev local (`docker-compose.yml` y `vite dev`) no se toca: el browser está en `localhost`, el default `http://localhost:8000` del client funciona porque el backend dev expone `:8000` al host.
- `CORS_ORIGINS` en prod queda sin uso efectivo (mismo origen, sin preflight); se mantiene como opcional para escenarios futuros.

## Consecuencias

### Positivas
- Un único dominio público — encaja con el modelo "un servicio por dominio" de Coolify sin labels custom.
- Sin CORS, sin preflight `OPTIONS` — un round-trip menos por request.
- HTTPS termina en Coolify y la pieza interna `backend:8000` no se expone jamás a internet.
- `X-Request-ID` cruza el proxy y el middleware del backend lo respeta — la traza request_id se preserva extremo a extremo.
- Portable: si mañana cambiamos Coolify por otro PaaS o por k8s, el patrón nginx sigue funcionando sin cambios en la app.

### Negativas
- Un hop nginx más en cada request (~1 ms en LAN; despreciable).
- `proxy_read_timeout 60s` cubre el mock; en M3, cuando la inferencia real puede tardar más, hay que subirlo y/o introducir streaming. Riesgo conocido, mitigación trivial.
- `/api/*` queda reservado en el frontend — un asset accidentalmente bajo `/api/...` chocaría con el proxy. No hay assets así hoy y la convención está documentada.
- `VITE_API_BASE_URL` está baked en la imagen — para ambientes adicionales (staging) hay que rebuild. Aceptable: solo hay un demo.

## Alternativas consideradas

1. **Dos dominios + CORS.** Duplica certificado TLS, requiere `CORS_ORIGINS` configurado y propagación DNS adicional. Operacionalmente más caro para un demo de un solo equipo. Rechazada.
2. **Backend expuesto público con CORS abierto.** Aumenta superficie de ataque (rate-limit y auth tendrían que blindar el `/analyze` directamente expuesto). Rechazada para portfolio público.
3. **Routing path-based con Traefik labels en `docker-compose.prod.yml`.** Funciona pero amarra el repo a Coolify/Traefik. Rechazada por portabilidad.

## Referencias

- design doc sec D3 (frontend serving en nginx).
- [`Docs/decisions/0006-frontend-serving-nginx.md`](0006-frontend-serving-nginx.md) — la base que este ADR extiende.
- `backend/app/api/middleware.py` — `RequestIdMiddleware` que consume `X-Request-ID` propagado por el proxy.
