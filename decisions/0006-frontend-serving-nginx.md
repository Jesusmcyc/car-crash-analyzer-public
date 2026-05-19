# ADR 0006 — Frontend en producción: nginx-alpine multi-stage, no `vite preview`

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

El frontend Vite genera assets estáticos. En el container que va a Coolify hay que servirlos. Opciones consideradas: `vite preview` (server dev de Vite), `serve`/`http-server` (Node), nginx-alpine, Caddy.

## Decisión

Multi-stage Docker:
- Builder `node:20-alpine` corre `npm ci && npm run build`.
- Runtime `nginx:alpine` con `nginx.conf` propio — gzip, fallback SPA a `index.html`, cache headers (`immutable` para hashed assets, `no-cache` para `index.html`).

Imagen final ~25 MB.

## Consecuencias

### Positivas
- Estándar de la industria para servir SPAs estáticas.
- Configuración explícita de cache + gzip — crítico para un demo público.
- Imagen runtime mínima; sin Node ni `node_modules` en producción.
- nginx maneja conexiones concurrentes orders-of-magnitude mejor que un Node-server casual.

### Negativas
- Una pieza más de configuración (`nginx.conf`) que mantener.
- Cambios de SPA fallback rules requieren rebuild del image.

## Alternativas consideradas

1. **`vite preview`.** Pensado solo para verificar el build, no para producción; la documentación de Vite lo dice. Rechazada.
2. **Caddy.** TLS automático es atractivo, pero Coolify ya termina TLS; Caddy duplicaría responsabilidades. Rechazada.
3. **`serve` (Node).** Funciona, pero arrastra Node y `node_modules` al runtime image (~120MB+ vs ~25MB). Rechazada.

## Referencias
- design doc sec D3.
