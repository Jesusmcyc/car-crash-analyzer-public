# ADR 0016 — Backend image budget: torch CPU-only + cache redirection en Dockerfile

**Estado:** Aceptado
**Fecha:** 2026-05-04
**Autor:** Jesús Moreno

## Contexto

Después de cerrar M3 con todos los tests verdes localmente y push a `master`, una verificación oportunista de prod (`https://cca.imb-central.tech`) durante M4 reveló que **prod estaba sirviendo el walking skeleton de M1**, no el pipeline real de M3. La respuesta de `/api/health` no incluía el campo `detector_backend` que M2 introdujo, y `/api/analyze` devolvía polígonos sintéticos con `backend:"mock"` en 500 ms — síntomas inequívocos de M1 corriendo, no M3.

Investigando el deployment log de Coolify para el último push (`4859c4d`, cierre M3), el build completaba `pip install` pero moría con exit code 255 durante el step `#37 [backend] exporting to image / exporting layers`. Coolify hacía rollback a la última imagen sana — la de M1, anterior a la introducción de `ultralytics` y `transformers`. Ese mismo modo de falla había estado ocurriendo silenciosamente en cada push desde M2; ningún deploy de M2 ni de M3 había aterrizado realmente en prod.

Cuatro problemas concatenados se descubrieron al reproducir el build localmente con `docker compose -f docker-compose.prod.yml build backend`:

1. **Build OOM/disk-full en `exporting layers`.** `uv.lock` resolvía `torch==2.11.0` y `torchvision==0.26.0` desde el index PyPI, lo cual arrastraba como dependencias transitivas la stack completa de NVIDIA/CUDA: `nvidia-cublas` (423 MB), `nvidia-cudnn-cu13` (366 MB), `nvidia-cufft` (214 MB), `nvidia-cusolver` (200 MB), `nvidia-nccl-cu13` (196 MB), `triton` (188 MB), `nvidia-cusparselt-cu13` (169 MB), `nvidia-cusparse` (145 MB), `nvidia-cuda-nvrtc` (90 MB), y siete paquetes `nvidia-*` adicionales. La imagen final descomprimida pesaba ~5–6 GB. El builder de Coolify reventaba al exportar las layers comprimidas. El demo es CPU-only por diseño (ADR 0009), así que la stack CUDA es 100% peso muerto.

2. **`cv2 ImportError: libxcb.so.1`.** `ultralytics` declara `opencv-python` (full GUI build) como dependencia transitiva. `pip install` lo instalaba *encima* de `opencv-python-headless` que ya estaba en `pyproject.toml`. Ambos paquetes shippean el mismo directorio `cv2/` en site-packages; pip da ownership al último instalado. La imagen `python:3.11-slim` no trae `libxcb1`/`libGL1`, y `import cv2` reventaba en runtime con `ImportError: libxcb.so.1: cannot open shared object file`.

3. **`PermissionError` descargando `rtdetr-l.pt`.** El Dockerfile creaba el WORKDIR `/app` como `root` y luego switcheaba a `USER app` (creado con `--no-create-home`). Ultralytics descarga pesos al CWD antes de moverlos al cache, y el user `app` no podía escribir en `/app` (ni en `/app/.cache/models`, que se monta como volumen y hereda los permisos del mount point).

4. **`PermissionError at /home/app` descargando `facebook/sam2-hiera-tiny`.** HuggingFace Hub cachea por default en `~/.cache/huggingface`, expandido a `/home/app/.cache/huggingface`. El user `app` no tiene `$HOME` válido (creado con `--no-create-home`), así que `os.makedirs('/home/app/.cache/...', exist_ok=True)` reventaba con `Permission denied`.

Cada uno de estos cuatro fallos se descubrió secuencialmente reproduciendo el smoke local; ninguno habría sido evidente sin ese paso, porque los tests `pytest -m slow` corren en el host de desarrollo (Windows) que tiene `$HOME` válido, no usa el Dockerfile, y resuelve `cv2` desde el opencv-python-headless del venv local sin conflicto.

## Decisión

**Restringir `torch` y `torchvision` al index `https://download.pytorch.org/whl/cpu`** vía `[[tool.uv.index]]` + `[tool.uv.sources]` en `pyproject.toml`. Las wheels `+cpu` no traen ninguna dependencia `nvidia-*` ni `triton`. La imagen baja de ~5–6 GB a ~2.5 GB. El Dockerfile pasa `--extra-index-url https://download.pytorch.org/whl/cpu` a `pip install` para que pip pueda resolver las wheels `+cpu` (uv graba `torch==2.11.0+cpu` en `requirements.txt` exportado, pero pip por sí solo solo conoce PyPI).

**Remover `opencv-python` post-install y reinstalar `opencv-python-headless --force-reinstall --no-deps`** en el mismo `RUN` del Dockerfile. `pip uninstall opencv-python` borra el directorio `cv2/` aunque `opencv-python-headless` también declare ownership; el reinstall con `--force-reinstall --no-deps` restaura el `cv2/` desde la wheel headless sin re-resolver dependencias. El step termina con un smoke `python -c "import cv2; print('cv2 ok', cv2.__version__)"` que rompe el build si algo está mal.

**`chown -R app:app /app`** después del último `COPY`, antes del `USER app`. Combinado con `RUN mkdir -p /app/.cache/models/huggingface`, esto da al user `app` permisos de write tanto en el WORKDIR (necesario para los staging downloads de Ultralytics) como en el cache dir del volumen montado.

**Variables de entorno explícitas para los caches**, set al inicio del `ENV` block:
- `HOME=/app` — fija un `$HOME` coherente para el user sin home dir.
- `HF_HOME=/app/.cache/models/huggingface` y `HUGGINGFACE_HUB_CACHE=/app/.cache/models/huggingface` — redirige el cache de HuggingFace al volumen persistente.
- `XDG_CACHE_HOME=/app/.cache` — futuro-proof para cualquier lib que respete XDG.

Las cuatro correcciones viajan en un solo commit (`chore(backend): fit Coolify build budget with cpu-only torch + opencv/cache fixes`), porque ninguna por sí sola hace que el deploy funcione end-to-end.

## Consecuencias

**Positivas:**
- Imagen final cae de ~5–6 GB a 2.54 GB. El builder de Coolify exporta layers sin morir.
- Sin las wheels `nvidia-*`, el cold-start lee menos disco y consume menos RAM (relevante en VPS pequeños).
- Lock determinista mantiene la trazabilidad: `uv.lock` graba `torch==2.11.0+cpu` explícitamente.
- HF cache compartido con Ultralytics en el mismo volumen Docker, sobrevive redeploys.
- El smoke `import cv2` en el build catchea regresiones futuras (si alguien añade una dep que rearrastra opencv-python).

**Negativas:**
- El Dockerfile gana ~10 líneas de complejidad (extra-index-url, uninstall+reinstall opencv, chown, env vars). Documentado inline con comentarios cortos que apuntan a este ADR.
- Si se necesita GPU en el futuro (M5 fine-tuning local con CUDA, por ejemplo), hay que tener un Dockerfile.gpu separado o parametrizar el index. M5 corre en notebook fuera del repo, así que no afecta producción.
- Cuatro variables de entorno extra en `ENV`. Trivial pero hay que recordarlas si alguien debugga "¿por qué HuggingFace está cacheando aquí?".

**Operativas:**
- El primer cold-start en prod descarga ~80 MB RT-DETR-l + ~150 MB SAM 2 desde Ultralytics CDN + HuggingFace Hub. Toma 30–90s según red. El healthcheck `start_period: 120s` ya cubre eso (ADR 0009).
- Redeploys subsecuentes hit cache-warm (volumen `models-cache` persiste). Cold-start <10s.
- `/api/health` con `detector_backend:"rtdetr"` y `models_loaded:true` es la señal de "M3 real corriendo, listo para `/analyze`".

## Alternativas consideradas

- **`pip install --no-deps torch` y declarar manualmente todas las deps no-NVIDIA.** Frágil — cualquier minor bump de PyTorch puede mover deps; perdemos la garantía del lockfile.
- **Multi-stage con NVIDIA stack en un layer separado y borrado al final.** No funciona: `pip install` no permite "instalar y borrar selectivamente"; los `.so` quedan referenciados por torch.
- **Usar imagen base `pytorch/pytorch:2.x-cpu`.** Más grande de baseline (~1.5 GB vs ~120 MB de slim) y arrastra utilities que no usamos. La ganancia neta es nula.
- **Build de Coolify en un VPS más grande.** No ataca el root cause (image bloat) y cuesta dinero recurrente para desplazar un workload que cabe en 2.5 GB.
- **Copiar pre-fetcheado los pesos al image en build-time** (en lugar de descargar en runtime). Inflaria la imagen otros 230 MB; preferimos cold-start lento la primera vez con cache caliente persistente.
- **Para los permisos: cambiar a `useradd --create-home` y dejar HuggingFace cachear en `~/.cache/huggingface`.** Funciona pero el cache se pierde en cada redeploy (no es volumen). El fix elegido (HF_HOME al volumen) sobrevive redeploys.
- **Para opencv: instalar `libxcb1 libgl1 libglib2.0-0` en el apt-get step y dejar opencv-python full.** Añade ~10 MB al sistema base, pero deja la wheel duplicada (~70 MB de cv2/ shadowed). Net wash de tamaño y deja un foot-gun futuro (alguien podría usar GUI funcs por accidente).

## Notas para publicación

Este ADR documenta un ciclo de "el código verde local no es deploy verde en prod" que es genérico para cualquier stack PyTorch + ultralytics + transformers desplegado en CPU. Los cuatro fallos son reproducibles paso a paso y comunes para quien intente desplegar Ultralytics en un VPS modesto:

1. **Build OOM/disk** — pasa con cualquier image que arrastre torch CUDA por default.
2. **opencv-python double install** — pasa con cualquier proyecto que use ultralytics + slim images.
3. **WORKDIR no writable por non-root user** — pasa cada vez que ultralytics descarga assets en runtime.
4. **`$HOME` inválido para non-root sin `--create-home`** — pasa con cualquier librería HuggingFace en non-root containers.

Detectar todo esto sin un `docker build` local + smoke run es prácticamente imposible — los tests del pipeline corren contra el venv del host, no contra la imagen final. **Lección operacional para registrar:** el `make test` verde no implica `coolify deploy` verde; el quality gate "real" del milestone es un smoke `docker compose build backend && docker run` ejecutado al menos una vez antes de cerrar la fase.

## Referencias

- [`Docs/decisions/0009-cold-start-cpu.md`](0009-cold-start-cpu.md) — preload síncrono en lifespan; este ADR conserva el contrato.
- [`Docs/decisions/0008-rtdetr-vs-yolo.md`](0008-rtdetr-vs-yolo.md) — RT-DETR como detector default; sigue activo en CPU.
- [`Docs/decisions/0010-sam2-zero-shot.md`](0010-sam2-zero-shot.md) — SAM 2 hiera-tiny vía HuggingFace Hub; el cache redirect lo cubre.
- [PyTorch CPU index](https://download.pytorch.org/whl/cpu) — wheels sin CUDA.
- [HuggingFace `HF_HOME`](https://huggingface.co/docs/huggingface_hub/main/en/package_reference/environment_variables#hfhome) — variable que controla el root del cache.
- [Ultralytics download path](https://github.com/ultralytics/ultralytics/blob/main/ultralytics/utils/downloads.py) — descarga al CWD por default.
