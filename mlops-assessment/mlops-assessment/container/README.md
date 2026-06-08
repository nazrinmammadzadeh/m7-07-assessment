# Container Plan

## Base Image

`python:3.11-slim-bookworm` (Debian 12 slim variant)

**Rationale:**
- Official Python image with a published security patching cadence; CVE fixes land within days.
- Slim variant omits docs, manpages, and C compilers; ~55 MB base vs ~900 MB for the full image.
- Debian Bookworm packages `libgomp1` (OpenMP, the only LightGBM runtime dependency) in its standard repository.

**Alternative considered:** `gcr.io/distroless/python3-debian12`. Rejected because distroless removes `/bin/sh`, which breaks exec-based Kubernetes debugging (`kubectl exec`) and makes incident investigation significantly harder. The marginal size benefit (~10 MB) does not justify the ops cost.

---

## Bake vs Mount Decision: **BAKED IN**

The 45 MB `model.ubj` artefact is copied into the image at build time in the `model-fetcher` stage of the multi-stage Dockerfile. It is **not** mounted from a volume or downloaded at container startup.

| Criterion | Bake (chosen) | Mount |
|---|---|---|
| **Immutability** | ✅ Image SHA uniquely identifies model version | ❌ Container + volume must be tracked separately |
| **Rollback** | ✅ `kubectl set image` = model rollback in one command | ❌ Must coordinate image rollback + volume swap |
| **Cold start** | ✅ No download delay; model loads from local disk in ~8 s | ❌ Must wait for volume mount or model download |
| **Audit trail** | ✅ OCI label `com.acme-iot.model.version` is immutable in the image | ❌ Runtime volume contents can drift |
| **Image size** | ❌ +45 MB per model version in the registry | ✅ Shared volume across versions |
| **Registry coupling** | ❌ Internal model registry must be reachable at **build** time | ✅ Registry only needed at container startup |

**Decision:** Bake in. The 45 MB model is small; the per-version image overhead is trivial. Immutability and single-command rollback (see `runbooks/rollback.md`) outweigh the size cost. The CI/CD pipeline (`cicd/.github/workflows/deploy-model.yml`) rebuilds and pushes a new image on every model promotion, so model and image versions are always 1:1.

The `serving/capacity-plan.md` startup budget accounts for this: ~8 s for Python import + LightGBM model load, zero seconds for model download.

---

## Multi-Stage Build Summary

| Stage | Base | Purpose | Copied to runtime? |
|---|---|---|---|
| `builder` | `python:3.11-slim-bookworm` | Creates `/opt/venv` with all Python packages | `/opt/venv` only |
| `model-fetcher` | `python:3.11-slim-bookworm` | Downloads `model.ubj` from internal registry and verifies `sha256` | `model.ubj` only |
| `runtime` | `python:3.11-slim-bookworm` | Runs the FastAPI service as non-root `appuser` (UID 1001) | Final image |

No build tools (gcc, pip, curl) exist in the `runtime` stage.

---

## Image Size Estimate

| Layer | Size |
|---|---|
| `python:3.11-slim-bookworm` base | ~55 MB |
| `libgomp1` (apt) | ~1 MB |
| Python packages in `/opt/venv` (FastAPI, uvicorn, lightgbm, numpy, redis-py, prometheus-client, pydantic, structlog) | ~182 MB |
| `model.ubj` (LightGBM artefact) | ~45 MB |
| Application source `src/` | ~2 MB |
| **Total (uncompressed)** | **~285 MB** |
| **Total (compressed in GHCR)** | **~320 MB** |

---

## Image Registry and Tag Scheme

Registry: `ghcr.io/acme-iot/maintenance-predictor`

Tag scheme (defined in `lifecycle/model-registry.yaml`, enforced by `cicd/.github/workflows/deploy-model.yml`):

```
{env}-{YYYYMMDD}-{git-sha-8}
```

Examples:
- `ghcr.io/acme-iot/maintenance-predictor:prod-20260607-a1b2c3d4`
- `ghcr.io/acme-iot/maintenance-predictor:staging-20260607-e5f6a7b8`
- `ghcr.io/acme-iot/maintenance-predictor:stable` ← floating tag, updated post-production rollout

The `stable` tag is updated atomically after a successful full production rollout. It is used by the rollback runbook as a safe fallback if the previous versioned tag is not available.

---

## Security Notes

- **Non-root:** `appuser` (UID/GID 1001); process has no write access to `/app/model/`.
- **No shell in CMD:** `CMD` uses exec form; no shell spawned.
- **Pip cache:** `--no-cache-dir` prevents pip cache from persisting in layers.
- **Model verification:** `sha256sum -c model.ubj.sha256` runs in `model-fetcher` before copy; a corrupt or tampered model artefact fails the build.
- **Trivy scan:** CI runs `trivy image` with `--exit-code 1 --severity CRITICAL` before pushing (see `cicd/.github/workflows/deploy-model.yml`). `HIGH` CVEs open a GitHub issue but do not block.
- **Registry token:** fetched via Docker BuildKit secret (`--mount=type=secret,id=registry_token`); never embedded in an image layer.
