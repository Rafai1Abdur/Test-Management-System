# ADR-0008 — Embedding Model Registry; bge-m3 as MVP Default (Configurable)

- **Date:** 2026-09-02
- **Status:** Approved

## Context

We need multilingual EN/UR embeddings (dense + sparse for hybrid retrieval). A single good
model today (bge-m3) may be superseded; hard-coding a model would block swapping and
corrupt index consistency if swapped silently.

## Decision

1. **Registry-driven embeddings**: `embedding_model_registry` collection defines model id,
   provider/endpoint, vector size, supports (dense/sparse/colbert), languages, version,
   status, is_default.
2. **bge-m3 is the MVP default** (dense 1024 + sparse/lexical; strong EN/UR) served locally
   (vLLM/Ollama/ONNX) or via gateway — **not** a permanent hard-code.
3. Every Qdrant point and `rag_chunks.embedding` record stores `embedding_model_id` +
   `embedding_model_version`; Qdrant collections are versioned per embedding model
   (`chunks_v1_bge_m3`), alias-swapped.
4. Changing the embedding model is a formal **re-index event** (QDRANT.md §8) — never a
   silent router switch. Router pins the embed model to the live collection's registry row.

## Consequences

- **Positive:** model upgrades/rollbacks are safe and auditable; multi-tenant can run
  multiple model generations; eval can compare models (AI_EVALUATION.md).
- **Negative:** extra registry bookkeeping; collection naming must be disciplined.

## Invalidation triggers

A better evaluated multilingual model passing regression gates → still registry-driven
(update default + re-index).