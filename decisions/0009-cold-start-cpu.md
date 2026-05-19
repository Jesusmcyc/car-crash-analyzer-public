# ADR 0009 — Cold-start CPU: preload síncrono en lifespan + cache en volumen

**Estado:** Aceptado
**Fecha:** 2026-05-03
**Autor:** Jesús Moreno

## Contexto

M2 añade pesos reales al backend (~70 MB RT-DETR-l o ~22 MB YOLOv8s). Dos preguntas operativas:

1. **¿Cuándo se cargan los pesos?** Opciones: sync en startup vs lazy en primera request vs background con gate en `/health`.
2. **¿Dónde viven entre reinicios del container?** Sin persistencia, cada redeploy descarga desde el CDN de Ultralytics.

## Decisión

**Preload síncrono en `lifespan` startup de FastAPI.** Pesos cargados antes de aceptar tráfico; `/health` con `models_loaded: True` significa "puedo servir `/analyze` ahora".

**Cache en volumen Docker** (`models-cache:/app/.cache/models` en `docker-compose.prod.yml`, ya provisto desde M1). `os.environ["YOLO_CONFIG_DIR"] = settings.model_cache_dir` se setea al inicio del lifespan para que Ultralytics escriba ahí.

**Healthcheck Coolify `start_period: 120s`** durante el primer boot del VPS (cubre la descarga inicial). Tras validar empíricamente que la cache caliente reduce cold-start a <30s, se revierte a 90s con commit aparte que cita el dato medido.

**Si el preload lanza** (peso corrupto, red caída, OOM): `app.state.models_loaded = False`, `/health` reporta `status: "degraded"`, `/analyze` devuelve `503 INFERENCE_UNAVAILABLE` con `request_id`. El container queda vivo para que Coolify pueda inspeccionarlo.

## Consecuencias

### Positivas

- **Contrato de `/health` simple y único.** No hay estados intermedios `"starting"`. Coolify decide unhealthy/healthy con un solo bool.
- **Primera request rápida.** El usuario que prueba el demo no paga el cold-start; lo paga el container al bootear.
- **Reinicios baratos.** Tras el primer boot del VPS, el volumen mantiene los pesos. Boots subsiguientes < 30s.
- **Degraded mode auditable.** Si el lifespan falla, el container responde y se puede ver `curl /health | jq` qué pasó. Coolify marca unhealthy y notifica.

### Negativas

- **Container tarda 60–90s en aparecer healthy en primer boot del VPS.** Comparado con un container "starting fast", esto se siente lento. Mitigado por `start_period: 120s` durante esa ventana — Coolify ignora los healthcheck failures hasta que vence.
- **Si los pesos del CDN cambian de checksum sin invalidar cache, podemos servir versión desactualizada.** Riesgo bajo: Ultralytics versiona pesos por nombre de archivo (`rtdetr-l.pt` no cambia entre versiones).
- **Bump temporal de `start_period` requiere disciplina para revertir.** Documentado en commit del bump y en este ADR como follow-up.

## Alternativas consideradas

1. **Lazy loading en primera request.** Primera request paga 30–60s. UX pobre para un demo presentable. Rechazada.
2. **Background loading con gate en `/health`.** Container responde rápido, `/health` reporta `"starting"` hasta que termina. Añade complejidad: lock para evitar doble carga, gate en `/analyze`, semántica de tres estados en `/health`. Sin ganancia operativa proporcional para un demo CPU. Rechazada.
3. **Pesos baked en la imagen Docker.** Imagen final crece ~70MB; cada cambio de pesos requiere rebuild + redeploy. Pierde el principio "swap por env var" del ADR 0001. Rechazada.

## Referencias

- Spec maestro: design doc sec M2 (riesgos cold-start).
- Design decisions M2: design doc sec 2 D1 + D7.
- ADR 0007 — same-origin nginx proxy (`proxy_read_timeout`): [`0007-prod-connectivity-nginx-proxy.md`](0007-prod-connectivity-nginx-proxy.md).
- Implementación: `backend/app/main.py` (`lifespan`), `docker-compose.prod.yml` (`start_period`).
