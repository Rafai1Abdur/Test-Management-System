# Scalability

## 1. MVP stance

Modular monolith + Celery workers + MongoDB + Qdrant + object storage + Redis (ADR-0001),
single-server Docker Compose (ADR-0013). **No microservices, no Kubernetes** in MVP. The
objective is: do not over-engineer — but keep every component replaceable.

## 2. Horizontal scaling path (order of action)

1. **Stateless API** (already): `api` container xN behind Nginx load balancer (sticky not
   required; sessions in JWT/Redis). DB connection pooling tuned.
2. **Workers**: scale per-queue replicas (`worker-*`) whenever queue depth grows.
   `worker-embeddings` and `worker-ocr` are the natural GPU candidates if models move local.
3. **Redis**: standalone → sentinel/HA; used as broker only (+ small cache).
4. **MongoDB**: single → replica set (reads + failover) → shard by `school_id` if a cluster
   of schools outgrows one set; TTL/archival policies first (cheaper).
5. **Qdrant**: single → distributed mode with sharding by `school_id`; alias-based
   re-indexing stays identical.
6. **Storage**: MinIO single → multi-node/gateway → cloud S3; bucket lifecycle rules.
7. **Service extraction** (only when team-scale/stake warrants): split domain boundaries
   that are independently deployable:
   - `processing` (worker-ocr) → OCR service
   - `grading` (worker-grading) → grading service
   - `rag` → retrieval service (if external consumers emerge)
   - `export` → render service (bursty CPU)
   Each extraction keeps its MongoDB collections via domain-owned namespaces.

## 3. Scaling levers & design pressures

| Bottleneck | Levers (MVP) | Future |
|---|---|---|
| AI cost/latency | routing tier, model choice, batch | dedicated instances, caching |
| OCR throughput | worker-ocr count, queue prefetch | GPU inference |
| Retrieval latency | Qdrant payload indexes, cache hot scopes | Qdrant replicas |
| Mongo hot spots | indexes, TTL, rollups | replica set, sharding |
| Upload volume | streamed, presigned, worker-ocr | parallel upload service |

## 4. Read-heavy analytics strategy

- Aggregates precomputed in `analytics_cubes`/rollup collections by `q.analytics`
  (idempotent); dashboards read cubes, not raw.
- Point-in-time recalcs with backfill jobs; stale flag for transparency.

## 5. Data & domain invariants preserved during scaling

- Tenant filter remains mandatory at repository layer; Qdrant query filter remains
  mandatory even when sharded; evidence/audit remain global append-only.
- Idempotency keys, hash-chain audit, and versioning survive all topology changes
  (they are protocol-level, not process-level).

## 6. Operational guards

- Queue depth alerts; broker memory watch; connection pool monitors; per-user rate limits
  preserved when additional API nodes appear (Redis-backed counters remain central).
- Blue/green deploys for API + workers; alias swaps for Qdrant re-index; rolling workers
  (acks_late).