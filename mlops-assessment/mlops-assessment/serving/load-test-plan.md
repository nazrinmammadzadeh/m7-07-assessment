# Load Test Plan

## Objectives

1. **SLO validation** — confirm `POST /v1/predict` meets p95 ≤ 120 ms at 20 RPS sustained and 40 RPS burst (the thresholds in `serving/slos.yaml` `alert_latency_p95`).
2. **Replica sufficiency** — verify 2 replicas sustain 20 RPS with p99 < 200 ms, and 40 RPS burst without triggering the HPA.
3. **Break-point discovery** — find the RPS at which p95 first exceeds 120 ms with 2 replicas.
4. **HPA validation** — confirm that HPA scales from 2 → 3 replicas within 90 s when CPU exceeds 70 % at sustained 2× load.
5. **Batch endpoint** — confirm `POST /v1/predict/batch` (500 sensors) meets p95 ≤ 2,000 ms.

---

## Tool

**k6** (v0.50+). Scripts live in `tests/load/`. Run against the **staging** environment only.

---

## Scenario 1 — Steady-State Validation (Critical Path)

```javascript
// tests/load/steady_state.js
import http from "k6/http";
import { check } from "k6";

const TARGET = __ENV.TARGET || "https://api-staging.acme-iot.internal/maintenance-predictor";

function randomFeatureVector() {
  return {
    sensor_id: `SENS-${String(Math.floor(Math.random() * 5000)).padStart(5, "0")}`,
    feature_vector: {
      vibration_rms_24h: 0.5 + Math.random() * 4,
      vibration_peak_24h: 1.0 + Math.random() * 7,
      temp_mean_24h:      55 + Math.random() * 30,
      temp_std_24h:       0.3 + Math.random() * 3,
      fft_dominant_freq_hz: 40 + Math.random() * 120,
      vibration_trend_7d: (Math.random() - 0.5) * 0.6,
      temp_trend_7d:      (Math.random() - 0.5) * 0.3,
    },
  };
}

export const options = {
  stages: [
    { duration: "2m", target: 20 },   // ramp to 20 RPS
    { duration: "10m", target: 20 },  // hold at 20 RPS (sustained load)
    { duration: "2m", target: 0 },    // ramp down
  ],
  thresholds: {
    // SLO gate: p95 ≤ 120 ms — matches serving/slos.yaml alert_latency_p95
    "http_req_duration{endpoint:predict}": ["p(95)<120"],
    // Availability gate: < 0.1% errors — matches serving/slos.yaml api_availability
    "http_req_failed": ["rate<0.001"],
  },
};

export default function () {
  const res = http.post(
    `${TARGET}/v1/predict`,
    JSON.stringify(randomFeatureVector()),
    {
      headers: { "Content-Type": "application/json" },
      tags: { endpoint: "predict" },
    }
  );
  check(res, {
    "status 200":             (r) => r.status === 200,
    "has X-Model-Version":    (r) => r.headers["X-Model-Version"] !== undefined,
    "model version format":   (r) => /^(prod|staging)-\d{8}-[0-9a-f]{8}$/.test(
                                       r.headers["X-Model-Version"] || ""
                                     ),
  });
}
```

---

## Scenario 2 — Burst Test (2× Sustained)

```javascript
// tests/load/burst.js
export const options = {
  stages: [
    { duration: "2m", target: 20 },   // baseline
    { duration: "1m", target: 40 },   // ramp to burst (2× sustained = 40 RPS)
    { duration: "5m", target: 40 },   // hold burst
    { duration: "2m", target: 20 },   // recover
    { duration: "1m", target: 0 },
  ],
  thresholds: {
    "http_req_duration{endpoint:predict}": ["p(95)<120"],
    "http_req_failed": ["rate<0.01"],   // slightly relaxed during burst
  },
};
// (same default function as steady_state.js)
```

---

## Scenario 3 — Break-Point Discovery

```javascript
// tests/load/breakpoint.js
export const options = {
  stages: [
    { duration: "3m", target: 20  },
    { duration: "3m", target: 60  },
    { duration: "3m", target: 100 },
    { duration: "3m", target: 150 },
    { duration: "3m", target: 200 },
    { duration: "2m", target: 0   },
  ],
  thresholds: {
    // No hard pass/fail — we are observing, not gating
    "http_req_duration{endpoint:predict}": ["p(95)<1000"],
  },
};
```

Record the RPS at which p95 first exceeds 120 ms. Expected result: ≥ 150 RPS per 2-replica deployment (i.e., 4× the 40 RPS burst requirement).

---

## Scenario 4 — Batch Endpoint Validation

```javascript
// tests/load/batch.js
// Sends 10 concurrent batch requests of 500 sensors each over 5 minutes.
export const options = {
  vus: 2,
  duration: "5m",
  thresholds: {
    "http_req_duration{endpoint:batch}": ["p(95)<2000"],
    "http_req_failed": ["rate<0.001"],
  },
};

export default function () {
  const sensors = Array.from({ length: 500 }, (_, i) => ({
    sensor_id: `BATCH-${String(i).padStart(5, "0")}`,
    feature_vector: {
      vibration_rms_24h: 1.0 + Math.random() * 3,
      vibration_peak_24h: 2.0 + Math.random() * 5,
      temp_mean_24h:      60 + Math.random() * 20,
      temp_std_24h:       0.5 + Math.random() * 2,
      fft_dominant_freq_hz: 50 + Math.random() * 100,
      vibration_trend_7d: (Math.random() - 0.5) * 0.5,
      temp_trend_7d:      (Math.random() - 0.5) * 0.2,
    },
  }));

  const res = http.post(
    `${TARGET}/v1/predict/batch`,
    JSON.stringify({ sensors }),
    {
      headers: { "Content-Type": "application/json" },
      tags: { endpoint: "batch" },
      timeout: "10s",
    }
  );
  check(res, { "status 200": (r) => r.status === 200 });
}
```

---

## Success Criteria

| Criterion | Pass Threshold | Aligns with |
|---|---|---|
| p95 latency at 20 RPS (sustained) | ≤ **120 ms** | `serving/slos.yaml` `alert_latency_p95` |
| p95 latency at 40 RPS (burst) | ≤ **120 ms** | `serving/slos.yaml` `alert_latency_p95` |
| Error rate at 20 RPS | < 0.1 % | `serving/slos.yaml` `api_availability` |
| Error rate at 40 RPS | < 1.0 % | `serving/slos.yaml` `api_availability` (relaxed for burst) |
| Break-point (p95 > 120 ms) | ≥ **150 RPS** on 2 replicas | `serving/capacity-plan.md` |
| HPA scale-out time | ≤ 90 s from trigger | `serving/capacity-plan.md` HPA config |
| Batch p95 (500 sensors) | ≤ 2,000 ms | Batch endpoint SLA |

---

## Environment

- Target: **staging** (`https://api-staging.acme-iot.internal/maintenance-predictor`)  
- Image: `ghcr.io/acme-iot/maintenance-predictor:staging-{date}-{sha}` (same image promoted to prod)  
- Redis pre-populated with 5,000 synthetic critical-sensor feature vectors matching production schema  
- Load test is a **required gate** in `cicd/.github/workflows/deploy-model.yml` before the `deploy-production` job runs  
- Do **not** run break-point or burst tests against production

## Run Command

```bash
docker run --rm grafana/k6:0.50.0 run \
  -e TARGET=https://api-staging.acme-iot.internal/maintenance-predictor \
  --out json=results.json \
  tests/load/steady_state.js
```
