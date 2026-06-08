# ADR 0001 — Use Kafka + Flink for Critical-Sensor Stream Processing

**Date:** 2026-06-07  
**Status:** Accepted  
**Deciders:** ML Lead, Platform Lead, on-call representative  
**Consequences owner:** Platform Lead

---

## Context

50,000 sensors emit MQTT messages every 10 seconds (≈ 5,000 msg/s sustained). Approximately 5,000 sensors are designated "critical" and require failure-probability alerts delivered within 5 minutes of an anomalous reading. The model requires a 168-step rolling feature vector (7 days × 24 hourly aggregates) per sensor as input. Raw MQTT messages contain only the instantaneous vibration and temperature reading; the feature vector must be computed and maintained incrementally.

Three ingestion + stream-processing options were evaluated:

| Option | Ingestion | Feature computation | State backend |
|---|---|---|---|
| A — Kafka + Flink | Amazon MSK | Flink stateful operator | RocksDB |
| B — Kinesis + Lambda | Amazon Kinesis | Lambda + DynamoDB | DynamoDB |
| C — MQTT direct | MQTT Broker | In-process rolling buffer | In-memory |

---

## Decision

**Option A — Kafka + Flink.**

---

## Reasoning

### Stateful windowing without external round-trips

Flink maintains per-sensor state (the 168-step rolling window) in RocksDB, co-located with the Flink task. Each new event extends the window in-process and triggers inference without a database round-trip. Option B requires every Lambda invocation to read and write DynamoDB (≥ 5 ms per call × 5,000 events/s = significant cost and added latency). Option C keeps state in-memory inside the predictor but creates inconsistency across replicas and loses state on restart.

### Backpressure and burst handling

When the inference service is slow (e.g., during a model rollout), Flink's credit-based flow control queues events in Kafka without dropping them and without cascading pressure to the MQTT broker. Lambda + Kinesis lacks native per-record backpressure; shard limits and Lambda concurrency caps can cause silent drops under burst.

### Single ingestion bus for two consumers

The same Kafka topic (`sensor-telemetry`) is consumed independently by the Flink streaming job (consumer group `flink-stream`) and the Spark batch job (consumer group `spark-batch`). Kafka's offset model allows each consumer to read at its own pace with no coupling. Option B would require a separate Kinesis stream or S3 export for the batch path.

### Replay capability

Kafka retains 7 days of messages. After a Flink job failure (or a model rollout requiring backfill), the streaming job can seek to an earlier offset and reprocess without any upstream changes. DynamoDB (Option B) and in-memory (Option C) have no replay capability.

---

## Consequences

### Positive

- Both the streaming and batch paths share one ingestion layer; no duplicate broker infrastructure.
- Full replay capability for outage recovery and model backfill.
- Flink checkpointing to S3 every 60 s enables restart-from-checkpoint after task manager failure.

### Negative / Risks

- Amazon MSK adds ≈ USD 108/month (2 × `kafka.m5.large` brokers). This is included in the ≈ USD 280 monthly estimate in the root README.
- The team requires Flink operational expertise. Mitigation: deploy Flink on Amazon Kinesis Data Analytics (managed Flink) to reduce ops burden during the initial rollout.
- MSK cluster provisioning takes ~20 minutes; it must be included in the infrastructure bootstrap, not in the deployment pipeline.

### Neutral

- The Kafka topic `sensor-telemetry` uses 3 partitions per 1,000 sensors (150 partitions total). Partition count must be set at topic creation time; increasing it later requires a careful rebalance.
