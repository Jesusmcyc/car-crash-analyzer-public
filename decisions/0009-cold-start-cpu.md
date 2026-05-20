# ADR 0009 — CPU cold start: synchronous preload in lifespan + volume cache

**Status:** Accepted
**Date:** 2026-05-03
**Author:** Jesús Moreno

## Context

M2 adds real weights to the backend (~70 MB for RT-DETR-l or ~22 MB for YOLOv8s). Two operational questions:

1. **When are the weights loaded?** Options: synchronously at startup vs lazily on the first request vs in the background with a gate on `/health`.
2. **Where do they live between container restarts?** Without persistence, every redeploy downloads from the Ultralytics CDN.

## Decision

**Synchronous preload in the FastAPI `lifespan` startup.** Weights are loaded before accepting traffic; `/health` with `models_loaded: True` means "I can serve `/analyze` now".

**Cache in a Docker volume** (`models-cache:/app/.cache/models` in `docker-compose.prod.yml`, already provisioned since M1). `os.environ["YOLO_CONFIG_DIR"] = settings.model_cache_dir` is set at the start of the lifespan so Ultralytics writes there.

**Coolify healthcheck `start_period: 120s`** during the VPS's first boot (covers the initial download). Once it is empirically confirmed that a warm cache reduces the cold start to <30s, it is reverted to 90s in a separate commit that cites the measured figure.

**If the preload throws** (corrupt weights, network down, OOM): `app.state.models_loaded = False`, `/health` reports `status: "degraded"`, and `/analyze` returns `503 INFERENCE_UNAVAILABLE` with a `request_id`. The container stays alive so Coolify can inspect it.

## Consequences

### Positives

- **A simple, single `/health` contract.** There are no intermediate `"starting"` states. Coolify decides unhealthy/healthy with a single bool.
- **A fast first request.** The user trying the demo does not pay the cold start; the container pays it at boot.
- **Cheap restarts.** After the VPS's first boot, the volume keeps the weights. Subsequent boots are < 30s.
- **An auditable degraded mode.** If the lifespan fails, the container still responds, and `curl /health | jq` shows what happened. Coolify marks it unhealthy and notifies.

### Negatives

- **The container takes 60–90s to appear healthy on the VPS's first boot.** Compared to a container that "starts fast", this feels slow. Mitigated by `start_period: 120s` during that window — Coolify ignores healthcheck failures until it expires.
- **If the CDN weights change checksum without invalidating the cache, we may serve a stale version.** Low risk: Ultralytics versions weights by file name (`rtdetr-l.pt` does not change between versions).
- **A temporary `start_period` bump requires discipline to revert.** Documented in the bump commit and in this ADR as a follow-up.

## Alternatives considered

1. **Lazy loading on the first request.** The first request pays 30–60s. Poor UX for a presentable demo. Rejected.
2. **Background loading with a gate on `/health`.** The container responds fast, and `/health` reports `"starting"` until it finishes. This adds complexity: a lock to avoid double loading, a gate on `/analyze`, and three-state semantics on `/health`. No proportional operational gain for a CPU demo. Rejected.
3. **Weights baked into the Docker image.** The final image grows ~70MB; every weight change requires a rebuild + redeploy. This loses the "swap via env var" principle from ADR 0001. Rejected.

## References

- Master spec: design doc sec M2 (cold-start risks).
- M2 design decisions: design doc sec 2 D1 + D7.
- ADR 0007 — same-origin nginx proxy (`proxy_read_timeout`): [`0007-prod-connectivity-nginx-proxy.md`](0007-prod-connectivity-nginx-proxy.md).
- Implementation: `backend/app/main.py` (`lifespan`), `docker-compose.prod.yml` (`start_period`).
