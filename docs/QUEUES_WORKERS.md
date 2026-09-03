# Queues & Workers

## 1. Tech

**Celery + Redis broker** (ADR-0004). One Celery **application** with multiple named queues
(worker pools), loosened to separate worker processes/containers. Arq/RQ documented as
alternatives in ADR-0004; Celery chosen for maturity (retries, routing, inspect).

## 2. Queues & worker pools

| Queue | Consumption (domain work) | Triggers (examples) |
|---|---|---|
| `q.ingestion` | material extract/analyze/chunk | material upload, re-chunk |
| `q.embeddings` | embed + Qdrant index | chunks ready, re-index |
| `q.generation` | question/key/rubric generation | blueprint generate |
| `q.ocr` | page/region OCR + handwriting + translation | paper submit |
| `q.grading` | AI grading eval per answer | attempt ready |
| `q.export` | PDF/DOCX render, reports | publish/print, analytics refresh |
| `q.analytics` | rollups | finalization schedule |
| `q.system` | maintenance (orphan sweep, TTL, watchdog) | schedule |

Workers: separate containers `worker-*` in MVP; each pool can scale independently.

## 3. Job model

- Persistent state in `jobs` collection (source of truth); Celery task id ↔ `job_id`.
- Fields: status (`QUEUED|RUNNING|SUCCEEDED|FAILED|CANCELED`), attempts, max_attempts,
  last_error, retryable, progress 0–1, heartbeat, trace_id, timestamps.
- Progress: workers report per-stage `progress` (e.g., ingestion pipeline stages,
  OCR page indices, grading n/total).
- Failed jobs: `FAILED` + error code/message; retryable vs not; DLQ (`q.dead`) with alert.
- Webhooks/notifications on terminal states (configurable).

## 4. Idempotency

- Task keys derived from domain identity (`material_id:stage`, `attempt_id:question_id`,
  `exam_id:version:paper_id:format`). Re-dispatch does not duplicate effects:
  - MongoDB: create with unique `idempotency_key` (unique index) → skip-on-duplicate.
  - Qdrant: upsert by `chunk_id`/point id; regenerate deterministic.
  - Storage: content-hash object names → identical writes are no-ops.
- Celery `acks_late` + prefetch 1 for long tasks → safe retry on worker crash.

## 5. Retries & failure policy

- Retry: exponential backoff (base 30 s, factor 2, max 5 attempts default; per-task).
- Transient vs permanent classification: provider/network/queue errors → retryable;
  schema/validation/security errors → non-retryable.
- After max attempts → `FAILED(RETRYABLE)` + admin alert + DLQ.
- Watchdog: tasks `RUNNING` longer than `timeout×k` or heartbeat stale → requeue once,
  then manual review.

## 6. Backpressure & rate control

- Broker-level: concurrency limits per queue; per-provider semaphores in AI gateway;
  Redis-based token buckets for provider rate limits.
- Queue depth metrics → autoscale hooks (MVP: alert boundaries);
  admin pause/resume per queue.

## 7. Observability hooks

- Task latency bucket per queue/type; success/retry/failure ratios; mean attempt counts;
  queue depth/age gauges; heartbeat lag (see [OBSERVABILITY.md](OBSERVABILITY.md)).

## 8. Extraction boundaries

All task logic lives in worker entrypoints calling the same **domain services** as the API
(no logic duplication). When a service is later extracted (SCALABILITY.md), workers move
with their domain: `q.ocr` + `processing` domain, `q.grading` + `grading` domain, etc.