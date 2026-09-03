# ADR-0004 — Celery + Redis for Background Work

- **Date:** 2026-09-02
- **Status:** Approved

## Context

Long-running work (ingestion, OCR, embedding, generation, grading, rendering, analytics)
must not block HTTP. We need queues, worker pools, retries, progress, failure states.
Alternatives considered: Celery (+Redis), RQ, Arq, Dramatiq, custom asyncio loops.

## Decision

1. **Celery** + **Redis** broker/result backend (ADR-0004 has alternatives recorded).
2. One Celery app; named queues per domain pool (see QUEUES_WORKERS.md) and separate worker
   containers in MVP.
3. Persistent job state in MongoDB (`jobs`) as source of truth; `acks_late`, prefetch 1,
   exponential-backoff retries with max attempts, DLQ + alerts; idempotent task keys.
4. Watchdog for stale `RUNNING` tasks.

## Consequences

- **Positive:** mature, documented, supports routing/priorities/beat; huge ecosystem;
  Python-native.
- **Negative:** heavier than Arq for pure-async; Redis adds a dependency (already needed for
  rate limiting/cache); Celery has a learning curve re: task design.

## Invalidation triggers

Overhead outweighs benefits at scale for specific pools → extract those pools to plain
pipelines (e.g., Kafka) while keeping the `jobs` state contract stable.