# Qdrant — The Derived Semantic Knowledge Index

## 1. Role and terminology

Qdrant hosts the **Derived Semantic Knowledge Index**: vector and search representations
*derived from* MongoDB facts + object-storage artifacts. It is:

- **NOT the system of record** — MongoDB is (ADR-0003).
- **NOT merely a cache** — it is a first-class component with index schemas, payload
  semantics, and its own lifecycle/backup strategy. But its contents are **always
  rebuildable** from MongoDB (`rag_chunks`, documents, questions) + object storage.
- **Not a raw document store** — it stores points whose payloads reference canonical records
  by ULID (`chunk_id` = `rag_chunks.public_id`, `question_id` = `questions.public_id`).

**Rebuild contract (invariant #9):** for any collection version, there exists a replay job
`embed → index` that exactly reproduces the index from MongoDB + storage. Rebuilds are safe
to run while serving (dual-write to new collection, then atomic alias swap).

## 2. Collections (naming: `<type>_v<N>_<embedding_model_id>`)

| Collection | Content | Vector config | Payload keys |
|---|---|---|---|
| `chunks_v1_bge_m3` (default) | Curriculum chunks | dense `bge-m3` 1024-d named `dense`; sparse named `sparse` | chunk metadata (see below) |
| `questions_v1_bge_m3` | Approved question bank items (semantic retrieval) | dense + sparse | question metadata, public_id |
| `grading_context_v1_bge_m3` (optional Phase 8+) | Approved materials usable as grading context | dense | same as chunks |

The `_v<N>_<model>` suffix makes **embedding-model/version changes additive**: a new
embedding model → register in `embedding_model_registry` → build `chunks_v2_<model>` →
alias swap → deprecate old. Rollback = alias swap back. No in-place vector migration.

## 3. Point schema (chunks collection)

```
id           = ULID (rag_chunks.public_id)
vectors      = { dense: [1024 floats], sparse: {indices:[...], values:[...]} }
payload      = {
  "chunk_id": ULID, "material_id", "school_id", "academic_year_id",
  "subject_id", "chapter_id", "section_id", "topic_id": null,
  "grade", "language", "content_type", "source_type", "authority",
  "page_number", "learning_objective", "difficulty",
  "embedding_model_id", "embedding_model_version", "embedded_at"
}
```

## 4. Payload indexes (filter-first retrieval)

Create `INDEX` filters on the payload fields used in scoped retrieval:
`school_id`, `academic_year_id`, `subject_id`, `chapter_id`, `section_id`, `topic_id`,
`grade`, `language`, `authority`, `content_type`, `material_id` (list field — Qdrant
`Integer`/`Keyword` list indexes), `embedding_model_id`.

Rules:
- **Every query filters on `school_id` at minimum** (tenant isolation at the index layer).
- Authority handling: exam generation filters `authority IN [PRIMARY, SECONDARY]` by default,
  or restricts to explicit `material_id[]` allowlist from the blueprint.
- `topic_id` nullable — use `IS NULL` semantics where permitted, or store empty string sentinel
  (avoid payload schema drift).

## 5. Embedding strategy

- Default MVP model: **`bge-m3`** (multilingual EN/UR, dense 1024 + sparse/lexical output,
  good zero-shot for Urdu) — served locally (vLLM/Ollama/ONNX) or via gateway provider.
- **Registry-driven** (ADR-0008): selection is configuration + `embedding_model_registry`,
  never hard-coded. All points and `rag_chunks.embedding` records carry
  `embedding_model_id` + version.
- Text-to-embed: `title+section context + chunk text` (dense), raw chunk tokens (sparse).
  Store the model fingerprint in payload for drift detection.

## 6. Retrieval (hybrid pipeline)

Implemented in the `rag` domain, detailed in [RAG.md](RAG.md):

```
query → build tenant/scope filter → dense search (filter) → sparse search (filter)
     → hybrid fusion (RRF) → optional rerank (cross-encoder or LLM rerank)
     → top-k points → resolve to rag_chunks via chunk_id → context assembly
```

## 7. Collection lifecycle

| Stage | Actions |
|---|---|
| Create | `create_collection` with dense + sparse named vectors; payload indexes |
| Build | Embed worker streams chunks in batches (idempotent by `chunk_id`) |
| Activate | Alias `chunks_live` → new collection; readers use alias only |
| Update | Only via new collection + alias swap (never in-place mutation) |
| Deprecate | Old collection kept for N days (rollback window), then deleted |
| Delete | Only after confirm via rollback drill on restore-from-backup |

## 8. Re-indexing & versioning

- **Re-embed (same model, changed chunk text):** upsert changed `chunk_id`s; tombstone
  removed chunks (delete point after confirming rag_chunks no longer `is_active`).
- **New embedding model:** registry row → new collection → replay → alias swap → old
  deprecated. Dashboards show `last indexed at` + per-model coverage.
- Versioning is per-point (`embedding_model_version`) and per-collection (`_v<N>`).

## 9. Backup & recovery

- Qdrant snapshots are **fast-recovery aids only** — the index is fully rebuildable.
- Backup strategy: take Qdrant snapshot (consistent via WAL) alongside MongoDB backup;
  restore drill on a completed alias swap (see [BACKUP_RECOVERY.md](BACKUP_RECOVERY.md)).
- MinIO/MongoDB restore → rebuild index if Qdrant backup is unusable (RPO/RTO impact
  documented there).

## 10. Scaling

- MVP: single Qdrant node on Docker Compose (ADR-0013).
- Growth path: Qdrant distributed mode (sharding by `school_id` via `shard_key`), read
  replicas for retrieval-heavy workloads; the `rag` domain API hides topology.
- Monitoring: collection size, point counts per school, query latency p95, failed points,
  reindex coverage (see [OBSERVABILITY.md](OBSERVABILITY.md)).

## 11. Interaction rules (enforced)

- Backend never writes to Qdrant except through the `rag` indexing service.
- Qdrant is never queried without the built scoped filter.
- Qdrant payloads never contain the full student answers or raw PII.