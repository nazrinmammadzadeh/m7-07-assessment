# Architecture Justification

## Pattern Chosen: Lambda-Lite (Stream for Critical Path + Daily Batch for All Sensors)

### Problem framing

The workload has two fundamentally different latency requirements:

- **Daily report** for schedulers: hours of latency acceptable; correctness matters more than freshness.
- **Near-real-time alerts** for critical sensors: 5-minute end-to-end budget; correctness and freshness both matter.

A single architecture pattern cannot serve both efficiently. Three candidates were evaluated:

| Pattern | Critical-path latency | Batch cost | Complexity |
|---|---|---|---|
| **Pure batch** (hourly Spark) | 30–60 min worst case | Low | Low |
| **Pure streaming** (Flink for all 50k) | < 5 min | High — 5,000 events/s inference | High |
| **Lambda-Lite** (streaming for 5k critical + batch for all 50k) | < 5 min | Low | Medium |

Pure batch fails the critical-path SLO. Pure streaming runs LightGBM inference at 5,000 events/s (all sensors every 10 s), which is both unnecessary and expensive for sensors whose predictions are only consumed in daily reports.

**Lambda-Lite** routes only the ~10% of sensors that are "critical" through the streaming path, reducing real-time inference load by 90% while satisfying both SLOs.

### Why Kafka as the central bus

Kafka decouples the MQTT broker from all downstream consumers. The Flink job and the Spark batch job each have their own consumer group and read at their own pace. Kafka's 7-day log retention enables:

- Replay after a Flink job crash or checkpoint corruption.
- Backfill when a new model version is promoted and we want to re-score the last week of data.
- Forensic investigation of anomalous predictions by correlating message offsets with inference logs.

Alternative considered: AWS Kinesis. Rejected because Kinesis default retention is 24 hours (extendable at extra cost), its shard model complicates the 50,000-sensor fan-out, and it cannot serve the Spark batch consumer without a separate S3 export.

### Why Redis as the feature store

The critical path requires a feature lookup before every inference call. Redis delivers sub-millisecond GET operations, which fits inside the 5 ms Redis budget (see `serving/capacity-plan.md`). Alternatives:

- **Feast on DynamoDB**: adds ≥ 15–20 ms per lookup, threatening the 120 ms HTTP SLO.
- **In-memory cache in the predictor**: inconsistent across replicas; eviction under bursty load.

Redis ElastiCache with 2 nodes (primary + replica) provides HA without the latency penalty.

### Why LightGBM (see also ADR 0002)

The 45 MB model artefact is small enough to bake into the container image. This means every image tag uniquely identifies a model version, simplifying rollback to a single `kubectl set image` command (see `runbooks/rollback.md`). A Transformer model at 400 MB+ would make baked images impractical and necessitate a runtime model-mount strategy, which decouples image and model lifecycles and complicates the rollback procedure.

## Trade-offs Accepted

| Trade-off | Consequence | Accepted because |
|---|---|---|
| Non-critical sensors receive no real-time alerts | A non-critical sensor could fail unexpectedly | Business requirement states only critical sensors need < 5 min alerting |
| LightGBM caps model expressiveness vs. a Transformer | Complex long-range temporal patterns may be missed | FFT + rolling-stat features proxy most patterns; AUC gate (≥ 0.82) enforces quality floor |
| Model baked into image | Image rebuild required on every model update | CI/CD pipeline automates rebuilds; baking guarantees immutability and fast rollback |
| Redis is stateful, single-region infrastructure | Redis cluster failure blocks the critical path | Mitigated by ElastiCache Sentinel; /readyz probe fails fast; alert fires within 1 min |
| Lambda-Lite, not full Lambda | No redundant batch path to validate streaming results | Batch path results are the ground truth for schedulers; streaming alerts are advisory |

## Scalability Ceiling

The current design supports ~40 RPS burst on the critical path with 4 replicas. If the critical-sensor count grows beyond ~20,000 (from 5,000 today), the Flink job will produce > 2,000 inference calls per second, which requires horizontal scaling of the predictor service. At that point, introducing a Kafka topic for inference requests (vs. direct Flink → HTTP calls) is the recommended next step, decoupling the Flink job from predictor availability.
