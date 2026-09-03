# Observability

## 1. Layers

| Layer | Tooling plan | MVP |
|---|---|---|
| Structured logs | JSON logs (Python `logging` + JSON formatter), request IDs | API + workers → Loki/ELK or simple files → grep |
| Metrics | Prometheus `/metrics` (FastAPI + Celery + custom) | Prometheus + Grafana |
| Traces | OpenTelemetry (FastAPI async, Celery, DB spans) | OTEL collector → Jaeger (optional MVP) |
| Dashboards/alerting | Grafana; Alertmanager; thresholds below | yes |

## 2. Instrumented surfaces

**API:** per-endpoint latency histograms, error rates by code class, active requests,
rate-limit hit counters.

**Database:** Mongo connection pool (available/checked-out), command latency, cursor/scan
counter, per-collection document counts (coverage of indexes).

**Queue/workers:** queue depth by name, age of oldest message, task duration by type/queue,
success/retry/failure ratios, attempt count distribution, heartbeat lag, dead-letter size.

**AI subsystem:** model latency by provider/model/capability, token usage (in/out),
estimated cost by task, **AI failures** by cause (timeout, schema, policy, quota), RAG:
retrieval latency, candidate counts per leg, fusion/rerank latency, retrieval errors,
**RAG retrieval quality proxies**: teacher edit rate on generated content,
feedback flags.

**Grading:** grading confidence distribution, **teacher override rate** (delta between
candidate and final), mean absolute error vs candidate, time-to-review, queue size by reason.

**Coverage/scope (generation):** chapter-weighting deviation distribution vs blueprint
weights (± tolerance), out-of-scope candidate rejections (expected 0), coverage-lock events.

**Storage:** bucket sizes/object counts, presign issuance rate, upload/download throughput,
orphan sweep results.

**Infra:** CPU/mem/disk of containers, volume usage (critical for WSL2/named volumes),
backup job outcomes.

## 3. Key alert thresholds (initial)

| Alert | Condition |
|---|---|
| API p95 error rate | > 1% over 5 min |
| OCR failure rate | > 5% jobs failing |
| Grading low-confidence spike | queue items > threshold or override rate > 30% |
| AI cost anomaly | daily cost > 20% above 7-day rolling average |
| Queue depth | any queue > N×workers for 5 min |
| Qdrant rebuild drift | `SUPERSEDED`/missing points ratio > 1% |
| Backup failure | any scheduled backup failure |

## 4. Privacy-aware telemetry

- Logs exclude PII and answer content (structured fields carry entity refs, not values).
- Evidence bundles (which do contain content refs) are access-controlled; metrics never echo
  text.

## 5. Dashboards (initial set)

1. **API & infra** — request volume, latency, errors, resource usage.
2. **AI Health** — model latency/cost/usage, eval-gate status.
3. **RAG** — search latency, cohort coverage, index version map.
4. **Grading** — verification queue, override rate, confidence calibration plots.
5. **Workers** — active tasks, queue age/depth, fail/retry.
6. **Storage & backups** — capacity trend, backup success, age of last good backup.

## 6. Standard library

- `observability.record(event, metrics_map, context)` helper used by domains;
  stratify by tenant for platform-level reporting while preserving school isolation.