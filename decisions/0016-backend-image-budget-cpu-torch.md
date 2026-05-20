# ADR 0016 — Backend image budget: CPU-only torch + cache redirection in the Dockerfile

**Status:** Accepted
**Date:** 2026-05-04
**Author:** Jesús Moreno

## Context

After closing M3 with every test green locally and pushing to `master`, an opportunistic prod check (`https://cca.imb-central.tech`) during M4 revealed that **prod was serving the M1 walking skeleton**, not the real M3 pipeline. The `/api/health` response did not include the `detector_backend` field that M2 introduced, and `/api/analyze` was returning synthetic polygons with `backend:"mock"` in 500 ms — unmistakable symptoms of M1 running, not M3.

Investigating the Coolify deployment log for the latest push (`4859c4d`, M3 close), the build completed `pip install` but died with exit code 255 during the `#37 [backend] exporting to image / exporting layers` step. Coolify rolled back to the last healthy image — the M1 one, predating the introduction of `ultralytics` and `transformers`. That same failure mode had been occurring silently on every push since M2; no M2 or M3 deploy had actually landed in prod.

Four chained problems were discovered when reproducing the build locally with `docker compose -f docker-compose.prod.yml build backend`:

1. **Build OOM/disk-full on `exporting layers`.** `uv.lock` resolved `torch==2.11.0` and `torchvision==0.26.0` from the PyPI index, which pulled in the full NVIDIA/CUDA stack as transitive dependencies: `nvidia-cublas` (423 MB), `nvidia-cudnn-cu13` (366 MB), `nvidia-cufft` (214 MB), `nvidia-cusolver` (200 MB), `nvidia-nccl-cu13` (196 MB), `triton` (188 MB), `nvidia-cusparselt-cu13` (169 MB), `nvidia-cusparse` (145 MB), `nvidia-cuda-nvrtc` (90 MB), and seven additional `nvidia-*` packages. The final uncompressed image weighed ~5–6 GB. The Coolify builder blew up while exporting the compressed layers. The demo is CPU-only by design (ADR 0009), so the CUDA stack is 100% dead weight.

2. **`cv2 ImportError: libxcb.so.1`.** `ultralytics` declares `opencv-python` (the full GUI build) as a transitive dependency. `pip install` installed it *on top of* the `opencv-python-headless` that was already in `pyproject.toml`. Both packages ship the same `cv2/` directory in site-packages; pip gives ownership to whichever is installed last. The `python:3.11-slim` image does not include `libxcb1`/`libGL1`, and `import cv2` blew up at runtime with `ImportError: libxcb.so.1: cannot open shared object file`.

3. **`PermissionError` downloading `rtdetr-l.pt`.** The Dockerfile created the WORKDIR `/app` as `root` and then switched to `USER app` (created with `--no-create-home`). Ultralytics downloads weights to the CWD before moving them to the cache, and the `app` user could not write to `/app` (nor to `/app/.cache/models`, which is mounted as a volume and inherits the permissions of the mount point).

4. **`PermissionError at /home/app` downloading `facebook/sam2-hiera-tiny`.** HuggingFace Hub caches by default in `~/.cache/huggingface`, which expands to `/home/app/.cache/huggingface`. The `app` user has no valid `$HOME` (created with `--no-create-home`), so `os.makedirs('/home/app/.cache/...', exist_ok=True)` blew up with `Permission denied`.

Each of these four failures was discovered sequentially while reproducing the local smoke test; none would have been evident without that step, because the `pytest -m slow` tests run on the development host (Windows), which has a valid `$HOME`, does not use the Dockerfile, and resolves `cv2` from the local venv's opencv-python-headless without any conflict.

## Decision

**Restrict `torch` and `torchvision` to the `https://download.pytorch.org/whl/cpu` index** via `[[tool.uv.index]]` + `[tool.uv.sources]` in `pyproject.toml`. The `+cpu` wheels carry no `nvidia-*` or `triton` dependency. The image drops from ~5–6 GB to ~2.5 GB. The Dockerfile passes `--extra-index-url https://download.pytorch.org/whl/cpu` to `pip install` so that pip can resolve the `+cpu` wheels (uv records `torch==2.11.0+cpu` in the exported `requirements.txt`, but pip on its own only knows about PyPI).

**Remove `opencv-python` post-install and reinstall `opencv-python-headless --force-reinstall --no-deps`** in the same `RUN` step of the Dockerfile. `pip uninstall opencv-python` deletes the `cv2/` directory even though `opencv-python-headless` also claims ownership; the reinstall with `--force-reinstall --no-deps` restores `cv2/` from the headless wheel without re-resolving dependencies. The step ends with a smoke test `python -c "import cv2; print('cv2 ok', cv2.__version__)"` that breaks the build if anything is wrong.

**`chown -R app:app /app`** after the last `COPY`, before the `USER app`. Combined with `RUN mkdir -p /app/.cache/models/huggingface`, this gives the `app` user write permissions both in the WORKDIR (needed for Ultralytics' staging downloads) and in the cache dir of the mounted volume.

**Explicit environment variables for the caches**, set at the start of the `ENV` block:
- `HOME=/app` — pins a coherent `$HOME` for the user with no home dir.
- `HF_HOME=/app/.cache/models/huggingface` and `HUGGINGFACE_HUB_CACHE=/app/.cache/models/huggingface` — redirect the HuggingFace cache to the persistent volume.
- `XDG_CACHE_HOME=/app/.cache` — future-proofs any library that respects XDG.

The four fixes travel in a single commit (`chore(backend): fit Coolify build budget with cpu-only torch + opencv/cache fixes`), because none of them on its own makes the deploy work end-to-end.

## Consequences

**Positives:**
- The final image drops from ~5–6 GB to 2.54 GB. The Coolify builder exports layers without dying.
- Without the `nvidia-*` wheels, the cold start reads less disk and consumes less RAM (relevant on small VPSes).
- A deterministic lock preserves traceability: `uv.lock` records `torch==2.11.0+cpu` explicitly.
- The HF cache is shared with Ultralytics on the same Docker volume and survives redeploys.
- The `import cv2` smoke test in the build catches future regressions (if someone adds a dependency that pulls opencv-python back in).

**Negatives:**
- The Dockerfile gains ~10 lines of complexity (extra-index-url, opencv uninstall+reinstall, chown, env vars). Documented inline with short comments that point to this ADR.
- If GPU is needed in the future (local M5 fine-tuning with CUDA, for example), a separate Dockerfile.gpu is required or the index must be parameterized. M5 runs in a notebook outside the repo, so it does not affect production.
- Four extra environment variables in `ENV`. Trivial, but they have to be remembered if someone debugs "why is HuggingFace caching here?".

**Operational:**
- The first cold start in prod downloads ~80 MB of RT-DETR-l + ~150 MB of SAM 2 from the Ultralytics CDN + HuggingFace Hub. It takes 30–90s depending on the network. The `start_period: 120s` healthcheck already covers that (ADR 0009).
- Subsequent redeploys hit a warm cache (the `models-cache` volume persists). Cold start <10s.
- `/api/health` with `detector_backend:"rtdetr"` and `models_loaded:true` is the "real M3 running, ready for `/analyze`" signal.

## Alternatives considered

- **`pip install --no-deps torch` and manually declare all non-NVIDIA dependencies.** Fragile — any minor PyTorch bump can shift dependencies; we lose the lockfile guarantee.
- **Multi-stage with the NVIDIA stack in a separate layer, deleted at the end.** Does not work: `pip install` does not allow "install and selectively delete"; the `.so` files remain referenced by torch.
- **Use a `pytorch/pytorch:2.x-cpu` base image.** Larger baseline (~1.5 GB vs ~120 MB for slim) and pulls in utilities we do not use. The net gain is nil.
- **Run the Coolify build on a larger VPS.** Does not attack the root cause (image bloat) and costs recurring money to shift a workload that fits in 2.5 GB.
- **Copy pre-fetched weights into the image at build time** (instead of downloading at runtime). It would inflate the image by another 230 MB; we prefer a slow cold start the first time with a persistent warm cache.
- **For the permissions: switch to `useradd --create-home` and let HuggingFace cache in `~/.cache/huggingface`.** Works, but the cache is lost on every redeploy (it is not a volume). The chosen fix (HF_HOME pointing to the volume) survives redeploys.
- **For opencv: install `libxcb1 libgl1 libglib2.0-0` in the apt-get step and keep full opencv-python.** Adds ~10 MB to the base system, but leaves the duplicated wheel (~70 MB of shadowed `cv2/`). A net wash on size, and it leaves a future foot-gun (someone could use GUI functions by accident).

## Notes for publication

This ADR documents a "green code locally is not a green deploy in prod" cycle that is generic for any PyTorch + ultralytics + transformers stack deployed on CPU. The four failures are reproducible step by step and common for anyone trying to deploy Ultralytics on a modest VPS:

1. **Build OOM/disk** — happens with any image that pulls in torch CUDA by default.
2. **opencv-python double install** — happens with any project that uses ultralytics + slim images.
3. **WORKDIR not writable by a non-root user** — happens every time ultralytics downloads assets at runtime.
4. **Invalid `$HOME` for a non-root user without `--create-home`** — happens with any HuggingFace library in non-root containers.

Detecting all of this without a local `docker build` + smoke run is practically impossible — the pipeline tests run against the host venv, not against the final image. **Operational lesson to record:** a green `make test` does not imply a green `coolify deploy`; the "real" quality gate for the milestone is a `docker compose build backend && docker run` smoke test executed at least once before closing the phase.

## References

- [`Docs/decisions/0009-cold-start-cpu.md`](0009-cold-start-cpu.md) — synchronous preload in the lifespan; this ADR preserves the contract.
- [`Docs/decisions/0008-rtdetr-vs-yolo.md`](0008-rtdetr-vs-yolo.md) — RT-DETR as the default detector; still active on CPU.
- [`Docs/decisions/0010-sam2-zero-shot.md`](0010-sam2-zero-shot.md) — SAM 2 hiera-tiny via HuggingFace Hub; the cache redirect covers it.
- [PyTorch CPU index](https://download.pytorch.org/whl/cpu) — CUDA-free wheels.
- [HuggingFace `HF_HOME`](https://huggingface.co/docs/huggingface_hub/main/en/package_reference/environment_variables#hfhome) — the variable that controls the cache root.
- [Ultralytics download path](https://github.com/ultralytics/ultralytics/blob/main/ultralytics/utils/downloads.py) — downloads to the CWD by default.
