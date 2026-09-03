# ADR-0007 — Qdrant as the Derived Semantic Knowledge Index

- **Date:** 2026-09-02
- **Status:** Approved

## Context

The product mandates Qdrant for vector search and MongoDB as primary storage. Earlier
drafts described Qdrant loosely as a "cache". A cache implies disposable and low-integrity;
Qdrant is more than that — it is a real search index with its own schema, payload indexes,
and lifecycle. But it is **not** a system of record.

## Decision

Terminology and architecture: **Qdrant = the Derived Semantic Knowledge Index.**

- MongoDB (**system of record**) owns canonical records: `rag_chunks`, `questions`,
  `learning_materials`, etc.
- Object storage (ADR-0005) owns the **source artifacts** (books, papers, generated files).
- Qdrant stores **derived representations**: vectors + payloads whose `chunk_id`/`question_id`
  reference canonical ULIDs. Everything in Qdrant is **rebuildable** from MongoDB + storage
  via a replay job (`rag_chunks → embed → index`, alias-safe; QDRANT.md §8).
- Collection naming encodes version + embedding model (`chunks_v1_bge_m3`); readers use a
  `chunks_live` alias; new embedding models/create/rebuilds are additive.
- Tenant isolation applied at the query filter and sharding strategy (`school_id`).

## Consequences

- **Positive:** clarity in ops (backups = convenience, not critical-path for the index);
  clean embedding-model change path; exportable "repair" capability; Qdrant can be wiped
  and rebuilt safely.
- **Negative:** a rebuild job must exist and be tested (drill); slight duplication of chunk
  metadata in Mongo (deliberate, enables evidence + rebuild + auditing).

## Invalidation triggers

Qdrant removed/withdrawn for some reason; switch to another vector store (same "derived"
contract applies — good).