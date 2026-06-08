# Rollback Runbook — maintenance-predictor

**Audience:** On-call engineer  
**Estimated execution time:** 8–10 minutes  
**Last reviewed:** 2026-06-07

---

## Trigger Conditions

Initiate rollback if **any** of the following alerts are firing and have not self-resolved within **5 minutes**:

| Alert Name | Defined In | Condition |
|---|---|---|
| `AvailabilityBurnRateFast` | `monitoring/alerts.yaml` | 5xx error rate > 1.44 % (14.4× budget) over 1 h |
| `LatencyBurnRateFast` | `monitoring/alerts.yaml` | p95 HTTP latency burning at >14.4× rate over 1 h |
| `EndToEndAlertLatencyHigh` | `monitoring/alerts.yaml` | Pipeline p95 > **240 s** for 5 consecutive minutes |

Additionally consider rollback if **all** of the following are true simultaneously:
- A deployment completed within the last 2 hours
- A `ModelVersionMismatch` alert has been firing for > 10 minutes post-deployment (stalled rollout)

**Do NOT roll back for these alerts alone:**

| Alert | Reason |
|---|---|
| `ModelVersionMismatch` | Expected transiently during rolling deploy; wait 10 min for convergence |
| `FeatureDriftDetected` | Requires a retrain, not a rollback |
| `PredictionDistributionShift` | Investigate first; rollback only if root-caused to a bad model |
| `AvailabilityBurnRateSlow` | Monitor; page if it escalates to `AvailabilityBurnRateFast` |
| `LatencyBurnRateSlow` | Monitor; page if it escalates to `LatencyBurnRateFast` |
| `BatchJobFailed` | Infrastructure issue; does not require model rollback |

---

## Pre-Rollback Checks (≤ 3 minutes)

- [ ] **Confirm the alert is real.** Open Grafana: `https://grafana.acme-iot.internal/d/maintenance-predictor`. Verify the alert condition is visible in the dashboard and not a Prometheus scrape gap.

- [ ] **Check for an active deployment.** If a deployment is in progress, `kubectl rollout undo` is the first action (Step 1 below).
  ```bash
  kubectl rollout status deployment/maintenance-predictor -n maintenance-predictor
  ```

- [ ] **Identify the current image tag.**
  ```bash
  kubectl get deployment maintenance-predictor -n maintenance-predictor \
    -o jsonpath='{.spec.template.spec.containers[0].image}'
  # Example output: ghcr.io/acme-iot/maintenance-predictor:prod-20260607-a1b2c3d4
  ```

- [ ] **Identify the previous stable tag.** Check the GitHub Actions run before the current deployment:  
  `https://github.com/acme-iot/maintenance-predictor/actions`  
  Or use the `stable` floating tag (updated after every successful full rollout):
  ```bash
  docker manifest inspect ghcr.io/acme-iot/maintenance-predictor:stable \
    | jq -r '.manifests[0].platform'
  ```

- [ ] **Confirm the rollback image exists in GHCR.**
  ```bash
  docker manifest inspect ghcr.io/acme-iot/maintenance-predictor:${PREV_TAG}
  # Must return 200 — if it fails, use :stable tag instead
  ```

---

## Rollback Steps

### Step 1 — Abort any in-progress rolling deployment

```bash
kubectl rollout undo deployment/maintenance-predictor -n maintenance-predictor
```

Wait up to 60 seconds:

```bash
kubectl rollout status deployment/maintenance-predictor \
  -n maintenance-predictor --timeout=90s
```

If the undo succeeds and alerts resolve within 5 minutes, **stop here and move to Post-Rollback Actions**.

---

### Step 2 — Explicit rollback to a known-good image tag

Use this step if `rollout undo` targets the wrong version (e.g. two consecutive bad deployments) or if the undo itself fails.

```bash
# Replace with the actual previous good tag from pre-rollback checks
PREV_TAG="prod-20260606-b9c8d7e6"

kubectl set image deployment/maintenance-predictor \
  predictor=ghcr.io/acme-iot/maintenance-predictor:${PREV_TAG} \
  -n maintenance-predictor

kubectl rollout status deployment/maintenance-predictor \
  -n maintenance-predictor --timeout=120s
```

If the versioned tag is unavailable, fall back to the `stable` floating tag:

```bash
kubectl set image deployment/maintenance-predictor \
  predictor=ghcr.io/acme-iot/maintenance-predictor:stable \
  -n maintenance-predictor
```

---

### Step 3 — Verify the rollback

```bash
# 1. All pods must be running the rollback image
kubectl get pods -n maintenance-predictor \
  -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'
# Expected: all pods show the PREV_TAG image

# 2. Readiness probe must return 200 with correct model_version
curl -s https://api.acme-iot.internal/maintenance-predictor/readyz | jq .
# Expected: {"status":"ready","model_version":"prod-20260606-b9c8d7e6","redis_ping_ms":...}

# 3. Confirm X-Model-Version response header matches the rolled-back version
# (X-Model-Version is defined in api/openapi.yaml and monitored by ModelVersionMismatch alert)
curl -si https://api.acme-iot.internal/maintenance-predictor/v1/predict \
  -X POST -H "Content-Type: application/json" \
  -d '{"sensor_id":"ROLLBACK-VERIFY-001","feature_vector":{"vibration_rms_24h":1.5,"vibration_peak_24h":2.1,"temp_mean_24h":65.0,"temp_std_24h":1.0,"fft_dominant_freq_hz":75.0,"vibration_trend_7d":0.0,"temp_trend_7d":0.0}}' \
  | grep -i "X-Model-Version"
# Expected: X-Model-Version: prod-20260606-b9c8d7e6
```

---

### Step 4 — Confirm alerts have resolved

Wait 5 minutes after Step 3 and verify in Grafana / Alertmanager that:

- [ ] `AvailabilityBurnRateFast` is no longer firing
- [ ] `LatencyBurnRateFast` is no longer firing
- [ ] `EndToEndAlertLatencyHigh` is no longer firing
- [ ] `ModelVersionMismatch` shows a single version (the rolled-back version)

**If alerts persist after rollback:** the issue is not model-version-related. Escalate to the platform team via `#platform-oncall` on Slack. Do not attempt a second rollback without understanding the root cause.

---

## Post-Rollback Actions

- [ ] Downgrade the failed model version in MLflow to reflect the rollback:
  ```bash
  mlflow models set-model-version-tag \
    --model-name lgbm-predictive-maintenance \
    --version "${FAILED_VERSION}" \
    --key status --value "rolled-back"
  ```
  Also transition the stage to `Archived` via the MLflow UI or CLI.

- [ ] File an incident report within 24 hours:  
  Template: `https://wiki.acme-iot.internal/incidents/template`

- [ ] Open a GitHub issue tagging `@ml-lead` with:
  - Failed version tag (e.g. `prod-20260607-a1b2c3d4`)
  - Alert names that fired
  - Wall-clock time from deployment to rollback decision
  - Preliminary hypothesis

- [ ] **Do not re-promote the rolled-back model version** to production without a root-cause analysis and a fix verified in staging.

---

## Contacts

| Role | Slack | Email / PagerDuty |
|---|---|---|
| ML Lead (`ml-lead`) | `@ml-lead` in `#ml-platform-oncall` | ml-lead@acme-iot.internal |
| Ops Lead (`ops-lead`) | `@ops-lead` in `#ml-platform-oncall` | PagerDuty: Acme IoT Ops |
| Platform Team | `#platform-oncall` | PagerDuty: Acme IoT Platform |
