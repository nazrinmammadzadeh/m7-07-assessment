# Capacity Plan

All numbers here are **consistent with `serving/slos.yaml`** and **consistent with `container/README.md`** (model baked in, no download at startup).

---

## Target Workloads

| Path | Sustained Load | Burst Load | Latency Budget |
|---|---|---|---|
| Critical path — `POST /v1/predict` | **20 RPS** | **40 RPS** | p95 ≤ **120 ms** |
| Batch sync — `POST /v1/predict/batch` | ~1 req/min (avg 500 sensors/req) | — | p95 ≤ 2,000 ms |
| Async jobs — `POST /v1/jobs` | 1 job/day (50,000 sensors) | — | Complete within 8 h |

These figures match exactly the SLOs in `serving/slos.yaml` (`alert_latency_p95` objective and `end_to_end_alert_latency` objective).

---

## Latency Budget Breakdown — Critical Path (HTTP)

The p95 HTTP budget for `/v1/predict` is **120 ms**. Internal components consume:

| Component | Budget | Notes |
|---|---|---|
| Network (Flink → service ingress) | ≤ 5 ms | Same-region VPC |
| Redis GET (feature lookup) | ≤ 5 ms | ElastiCache sub-ms local, 5 ms for cache miss + replication lag |
| LightGBM `model.predict()` | ≤ 5 ms | Benchmarked p95 on `c6i.large`; see ADR 0002 |
| Response serialization (Pydantic) | ≤ 3 ms | |
| **Total internal** | **≤ 18 ms** | **102 ms headroom vs 120 ms SLO** |

The 102 ms headroom absorbs:
- Python GC pauses (typically < 10 ms)
- Occasional Redis replication lag during ElastiCache failover (≤ 30 ms)
- Uvicorn worker scheduling jitter

**Model startup time:** ~8 s (Python import + LightGBM `Booster.load_model('/app/model/model.ubj')`). No model download occurs at startup because the model is baked into the image. This is accounted for in the Kubernetes readiness probe: `/readyz` begins returning 200 only after load completes.

---

## Replica Sizing — Critical-Path Service

**Hardware per replica:** 2 vCPU, 4 GB RAM — AWS `c6i.large`

**Concurrency model:** uvicorn with `--workers 2` (2 processes) × async I/O.  
Each worker handles one blocking `model.predict()` call at a time (LightGBM is not async-safe); remaining capacity is async I/O (Redis, response). Effective concurrent requests per replica: **4** (2 workers × ~2 pipelined async steps before predict).

**Throughput per replica at p95 ≤ 18 ms:**
- Single-request cycle time: ~18 ms
- Max throughput (4 concurrent): 4 / 0.018 ≈ **222 RPS** (theoretical ceiling)
- At 70 % utilisation target: 222 × 0.70 ≈ **155 RPS per replica**

**Replicas required:**

| Load | Replicas needed | Configured |
|---|---|---|
| 20 RPS sustained | 20 / 155 ≈ 0.13 → 1 (HA floor: 2) | **2 min** |
| 40 RPS burst | 40 / 155 ≈ 0.26 → 1 (HA floor: 2) | 2 (no scale-out needed) |
| HPA max (safety ceiling) | — | **4 max** |

Two replicas comfortably serve 40 RPS burst with < 15 % utilisation. The HPA is configured to scale out at **70 % CPU or p95 latency > 80 ms** (leaving 40 ms before the SLO is breached). Scale-out target of 4 provides a 4× capacity buffer.

---

## HPA Configuration

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: maintenance-predictor-hpa
  namespace: maintenance-predictor
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: maintenance-predictor
  minReplicas: 2
  maxReplicas: 4
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_request_duration_p95_ms    # custom metric from Prometheus adapter
        target:
          type: AverageValue
          averageValue: "80"    # scale out at 80 ms — 40 ms before the 120 ms SLO
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Pods
          value: 1
          periodSeconds: 30
    scaleDown:
      stabilizationWindowSeconds: 300   # 5-min cooldown; avoids flapping
```

---

## Monthly Cost Estimate

| Resource | Specification | Est. Monthly Cost (USD) |
|---|---|---|
| 2× `c6i.large` (critical-path replicas, on-demand) | 2 vCPU / 4 GB | 61.76 |
| 1× `c6i.large` (batch inference worker) | 2 vCPU / 4 GB | 30.88 |
| Amazon MSK — 2 brokers | `kafka.m5.large` | 108.00 |
| ElastiCache — Redis, 2 nodes | `cache.r7g.large` | 48.00 |
| S3 (results + features, ~15 GB/month) | Standard storage | 0.35 |
| Data transfer (est. 100 GB/month) | Same-region + NAT | 9.00 |
| CloudWatch / Prometheus / Grafana Cloud | Monitoring tier | 22.00 |
| **Total** | | **≈ USD 280** |

*Matches the root README estimate. Spot instances for the batch worker would reduce cost by ~60 % (≈ USD 12 instead of USD 31), but spot interruptions during the daily 00:30 UTC job require a retry mechanism.*

---

## Batch Job Sizing

The daily async job (50,000 sensors) runs on the same `c6i.large` batch worker:
- LightGBM batch inference (50,000 rows): ~2 s (vectorised `model.predict()`)
- Redis bulk GET (50,000 feature vectors at ~512 bytes each): ~25 s
- S3 Parquet write (~25 MB output): ~5 s
- **Total wall-clock time: < 5 minutes**

This fits well within the 8-hour window between job trigger (00:30 UTC) and dashboard query (06:00 UTC), leaving capacity for retries on transient S3 or Redis failures.
