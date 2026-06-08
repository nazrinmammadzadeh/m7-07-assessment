# syntax=docker/dockerfile:1.6
# =============================================================================
# Predictive Maintenance Inference Service
# Model: lgbm-predictive-maintenance
# Bake strategy: model artefact baked in at build time (see container/README.md)
#
# Three stages:
#   builder       — installs Python deps into an isolated virtualenv
#   model-fetcher — pulls and verifies the model artefact (sha256)
#   runtime       — slim, non-root production image
#
# Build command (CI uses this; see cicd/.github/workflows/deploy-model.yml):
#   docker build \
#     --build-arg MODEL_VERSION=prod-20260607-a1b2c3d4 \
#     --build-arg GIT_SHA=a1b2c3d4 \
#     --build-arg BUILD_DATE=20260607 \
#     --build-arg MODEL_REGISTRY_URL=https://registry.acme-iot.internal/models \
#     -t ghcr.io/acme-iot/maintenance-predictor:prod-20260607-a1b2c3d4 \
#     -f container/Dockerfile container/
# =============================================================================

# =============================================================================
# Stage 1 — builder
# Installs Python packages into /opt/venv so no build tools leak to runtime.
# =============================================================================
FROM python:3.11-slim-bookworm AS builder

WORKDIR /build

# gcc required to compile some lightgbm / numpy extensions; not in runtime image
RUN apt-get update && apt-get install -y --no-install-recommends \
        gcc \
    && rm -rf /var/lib/apt/lists/*

RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

COPY requirements.txt .
RUN pip install --upgrade pip==24.0 \
    && pip install --no-cache-dir -r requirements.txt


# =============================================================================
# Stage 2 — model-fetcher
# Downloads and verifies the model artefact from the internal model registry.
# Verification happens here; the runtime image never loads an unverified model.
# MODEL_VERSION must match the versioning.scheme in lifecycle/model-registry.yaml:
#   {env}-{YYYYMMDD}-{git-sha-8}   e.g. prod-20260607-a1b2c3d4
# =============================================================================
FROM python:3.11-slim-bookworm AS model-fetcher

ARG MODEL_VERSION
ARG MODEL_REGISTRY_URL=https://registry.acme-iot.internal/models

RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /model

RUN --mount=type=secret,id=registry_token \
    TOKEN=$(cat /run/secrets/registry_token) && \
    curl -fsSL \
         -H "Authorization: Bearer ${TOKEN}" \
         "${MODEL_REGISTRY_URL}/lgbm-predictive-maintenance/${MODEL_VERSION}/model.ubj" \
         -o model.ubj && \
    curl -fsSL \
         -H "Authorization: Bearer ${TOKEN}" \
         "${MODEL_REGISTRY_URL}/lgbm-predictive-maintenance/${MODEL_VERSION}/model.ubj.sha256" \
         -o model.ubj.sha256 && \
    sha256sum -c model.ubj.sha256 && \
    echo "Model artefact verified: ${MODEL_VERSION}"


# =============================================================================
# Stage 3 — runtime
# Minimal surface area: no build tools, no shell scripts, non-root user.
# libgomp1 is the only runtime dependency added beyond the slim base (OpenMP for LightGBM).
# =============================================================================
FROM python:3.11-slim-bookworm AS runtime

# libgomp1: OpenMP runtime required by LightGBM at inference time
RUN apt-get update && apt-get install -y --no-install-recommends \
        libgomp1 \
    && rm -rf /var/lib/apt/lists/* \
    # Non-root user for runtime
    && groupadd --gid 1001 appuser \
    && useradd --uid 1001 --gid 1001 --no-create-home --shell /bin/false appuser

WORKDIR /app

# Copy virtualenv from builder (no pip, no gcc in final image)
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

# Copy verified model artefact — BAKED IN, not mounted at runtime.
# Image tag = model version. Rollback = image rollback. See runbooks/rollback.md.
COPY --from=model-fetcher /model/model.ubj /app/model/model.ubj

# Copy application source
COPY src/ /app/src/

# OCI image labels (used by container inventory tooling)
ARG MODEL_VERSION=unknown
ARG GIT_SHA=unknown
ARG BUILD_DATE=unknown
LABEL org.opencontainers.image.title="maintenance-predictor" \
      org.opencontainers.image.vendor="Acme IoT" \
      org.opencontainers.image.version="${MODEL_VERSION}" \
      org.opencontainers.image.revision="${GIT_SHA}" \
      org.opencontainers.image.created="${BUILD_DATE}" \
      com.acme-iot.model.name="lgbm-predictive-maintenance" \
      com.acme-iot.model.version="${MODEL_VERSION}"

USER appuser

EXPOSE 8080

# MODEL_VERSION is read at startup and embedded in every X-Model-Version response header.
# It must match the tag scheme: {env}-{YYYYMMDD}-{git-sha-8}
ENV MODEL_PATH=/app/model/model.ubj \
    MODEL_VERSION=${MODEL_VERSION} \
    WORKERS=2 \
    PORT=8080 \
    REDIS_URL=redis://redis-feature-store:6379/0 \
    LOG_LEVEL=info

# Liveness probe: simple HTTP check (not readiness — avoids killing pods during Redis hiccup)
HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
    CMD python -c \
        "import urllib.request; urllib.request.urlopen('http://localhost:8080/healthz')" \
        || exit 1

CMD ["python", "-m", "uvicorn", "src.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8080", \
     "--workers", "2", \
     "--log-level", "info", \
     "--no-access-log"]
