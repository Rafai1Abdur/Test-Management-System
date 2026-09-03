# ADR-0001 — Modular Monolith + Workers over Microservices

- **Date:** 2026-09-02
- **Status:** Approved
- **Decision owner:** Architecture phase (this docs package)

## Context

The system must scale from a single school on one server to "many schools + many students +
many AI jobs", with clean boundaries for future extraction. Full microservices add
operational cost (service mesh, multi-repo, distributed transactions, tracing) that is
unjustified before we know real bottlenecks.

## Decision

1. Build a **modular monolith**: one FastAPI application containing isolated domain modules
   (`auth, school, enrollment, curriculum, materials, processing, rag, questions, exams,
   grading, results, ai, audit`).
2. Add **dedicated Celery worker processes** per domain pool (ingestion/embeddings/
   generation/ocr/grading/export/analytics/system).
3. Domains communicate only through domain-service interfaces/DTOs; each owns its
   MongoDB collections; no cross-domain raw queries.
4. Scalability path documented in SCALABILITY.md (horizontal API, per-pool workers, then
   selective extraction).

## Consequences

- **Positive:** simple ops (one deployable), fastest delivery, transaction ease,
  clear future extraction seams, easy local development on Windows/WSL2.
- **Negative:** single-process blast radius (mitigated by workers + health + restart);
  requires import discipline to keep boundaries real.

## Invalidation triggers

Deploy-scale evidence of independent scaling needs (documented in SCALABILITY.md); a second
team wanting to own a slice; database hot-spots that demand separate deployment of a domain.