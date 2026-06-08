# Architecture

## System Context

Two consumer types drive the design:

- **Maintenance schedulers** — review a daily HTML/Parquet report listing predicted failures across all 50,000 sensors for the next 72 hours. Latency tolerance: hours.
- **On-call technicians** — receive PagerDuty / SMTP alerts within 5 minutes when a critical sensor's failure probability reaches or exceeds 0.75. Latency tolerance: seconds.

These two requirements are incompatible with a single architecture pattern. The system uses a Lambda-Lite approach: a streaming path for the ~5,000 critical sensors and a batch path for all 50,000 sensors once daily.

## Component Diagram

```mermaid
flowchart TD
    subgraph Sensors["Field — 50,000 Industrial Sensors"]
        S1[Group A: Vibration + Temp]
        S2[Group B: Vibration + Temp]
        SN[...]
    end

    subgraph Ingestion["Cloud Ingestion Layer"]
        MQTT_B["MQTT Broker\nHiveMQ Cloud / AWS IoT Core\nTLS, QoS 1"]
        KAFKA["Apache Kafka — Amazon MSK\nTopic: sensor-telemetry\n7-day retention · 3 partitions/1k sensors"]
    end

    subgraph Features["Feature Engineering"]
        FLINK["Flink Streaming Job\nRolling 168-step window per sensor\nRocksDB state backend\nWrites to Redis on each new reading"]
        SPARK_B["Spark Batch Job\n23:00 UTC daily\nBulk feature computation for all 50k sensors\nWrites cold features to Redis + S3"]
        FS[("Redis Cluster\nFeature Store\nElastiCache — 2 nodes\ncache.r7g.large")]
    end

    subgraph Inference["Inference Layer"]
        CRIT_SVC["Critical-Path Predictor\nFastAPI + uvicorn\n2 vCPU / 4 GB per replica\nModel baked in: /app/model/model.ubj\nImage: ghcr.io/acme-iot/maintenance-predictor:{env}-{date}-{sha}\n2–4 replicas (HPA)"]
        BATCH_JOB["Batch Inference Job\nPython / Spark\nTriggered at 00:30 UTC\nCalls POST /v1/jobs (async endpoint)\nResults → S3 Parquet"]
    end

    subgraph Outputs["Outputs"]
        ALERT_SVC["Alert Service\nPagerDuty + SMTP fan-out\nTriggered when failure_prob_72h ≥ 0.75"]
        RPT["Report Store\nS3: s3://acme-iot-data/batch-results/\nPartitioned by prediction_date"]
        DASH["Scheduler Dashboard\nQueries S3 at 06:00 UTC"]
    end

    subgraph Obs["Observability"]
        PROM["Prometheus\nScrapes /metrics on all services"]
        GRAF["Grafana\nDashboard: d/maintenance-predictor"]
        MLF["MLflow Registry\nModel versions, stages, lineage"]
    end

    S1 & S2 & SN -->|TLS MQTT every 10 s| MQTT_B
    MQTT_B --> KAFKA
    KAFKA -->|consumer group: flink-stream| FLINK
    KAFKA -->|consumer group: spark-batch| SPARK_B
    FLINK -->|SET sensor:{id}:features| FS
    SPARK_B -->|bulk MSET| FS
    FS -->|GET on critical sensor reading| CRIT_SVC
    FS -->|bulk GET for all 50k| BATCH_JOB
    CRIT_SVC -->|failure_prob ≥ 0.75| ALERT_SVC
    BATCH_JOB --> RPT --> DASH
    CRIT_SVC -->|/metrics| PROM
    BATCH_JOB --> PROM
    PROM --> GRAF
    CRIT_SVC -.->|version, stage| MLF
```

## Critical Path — Data Flow (target: p95 ≤ 300 s end-to-end)

| Step | Component | Budget |
|---|---|---|
| 1 | Sensor emits reading via MQTT | — |
| 2 | MQTT Broker → Kafka propagation | ≤ 15 s |
| 3 | Flink window update + Redis write | ≤ 30 s |
| 4 | Flink triggers HTTP POST `/v1/predict` for critical sensor | — |
| 5 | Redis feature lookup in service | ≤ 5 ms |
| 6 | LightGBM `model.predict()` | ≤ 5 ms (p95) |
| 7 | Response returned; if prob ≥ 0.75, alert event published | ≤ 3 ms |
| 8 | Alert Service fans out to PagerDuty + SMTP | ≤ 30 s |
| **Total** | | **≤ ~75 s typical · ≤ 300 s SLO** |

The HTTP p95 budget for the `/v1/predict` endpoint is 120 ms (defined in `serving/slos.yaml`), well inside the 300 s end-to-end window. The headroom absorbs Kafka lag spikes and Redis replication delay.

## Batch Path — Data Flow (target: report ready by 06:00 UTC)

| Step | Time (UTC) | Action |
|---|---|---|
| 1 | 23:00 | Spark batch job reads raw Kafka topic for last 24 h; computes 168-step feature vectors for all 50,000 sensors; writes to Redis and S3. |
| 2 | 00:30 | Batch Inference Job calls `POST /v1/jobs` (async endpoint); receives `job_id`. |
| 3 | 00:30–02:00 | Async job runs inference on 50,000 sensors using model baked in the container; results written to `s3://acme-iot-data/batch-results/{date}/`. |
| 4 | 06:00 | Dashboard queries S3 and renders the daily maintenance report. |

## Failure Modes and Mitigations

| Failure | Impact | Mitigation |
|---|---|---|
| Redis node failure | Critical path blocked | ElastiCache with Redis Sentinel; /readyz probe returns 503 immediately |
| Kafka broker failure | Ingestion stops; alerts delayed | MSK — 3 brokers, 3-way replication factor |
| Inference replica crash | Remaining replicas absorb load; HPA adds replica | minReplicas: 2; pod disruption budget |
| Flink job crash | Critical alerts delayed | Flink checkpointing every 60 s to S3; auto-restart on failure |
| Model version mismatch across replicas | `ModelVersionMismatch` alert fires | Alert resolves once rolling deploy completes (~3 min); runbooks/rollback.md if it persists |
