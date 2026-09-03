# Documentation Index — AI-Powered School Test & Examination Management System

This directory is the **authoritative specification** for the system. Future implementation
work (backend, frontend, infrastructure) MUST be traceable to these documents. If an
implementation decision disagrees with a document, the change must be recorded in a new
ADR before implementation.

## Repository status at the start of this phase

- The repository was **empty** (0 files, no `.git`) when inspected on 2026-09-02.
- This phase created **documentation only**. No application code, services, or databases exist.
- No Git commit has been created yet (intentional — pending approval).

## Documentation map

| Document | Purpose |
|---|---|
| [PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md) | Product vision, actors, workflows, MVP / Phase 2 / Future split |
| [REQUIREMENTS.md](REQUIREMENTS.md) | Functional & non-functional requirements; traceability to the 51 objectives |
| [ARCHITECTURE.md](ARCHITECTURE.md) | C4 architecture, modular-monolith domains, container topology, planned repo tree |
| [SYSTEM_DESIGN.md](SYSTEM_DESIGN.md) | End-to-end sequence/flow designs for every major workflow |
| [DATABASE.md](DATABASE.md) | MongoDB modeling philosophy, embed-vs-reference decisions, tenancy, retention |
| [MONGODB_SCHEMA.md](MONGODB_SCHEMA.md) | Full collection catalog: fields, types, indexes, validation, lifecycle |
| [QDRANT.md](QDRANT.md) | The **Derived Semantic Knowledge Index**: collections, vectors, payloads, lifecycle |
| [RAG.md](RAG.md) | Curriculum-aware retrieval: filters → dense + sparse → hybrid fusion → rerank → context |
| [CHUNKING.md](CHUNKING.md) | Hierarchical/semantic chunking (Document → Book → Chapter → Section → Topic → Concept) |
| [AI_ARCHITECTURE.md](AI_ARCHITECTURE.md) | AI capabilities, model registry, the "AI is never final authority" invariant |
| [AI_PROVIDER_GATEWAY.md](AI_PROVIDER_GATEWAY.md) | Provider adapter architecture (OpenAI, Gemini, Anthropic, GLM, Ollama, vLLM, ...) |
| [MODEL_ROUTING.md](MODEL_ROUTING.md) | Task → model routing policy (capability, cost, latency, language, vision) |
| [MATERIAL_INGESTION.md](MATERIAL_INGESTION.md) | Upload → validation → extraction → analysis → chunking → indexing → approval pipeline |
| [OCR.md](OCR.md) | OCR architecture: providers, confidence, machine-readable exams, printed text |
| [HANDWRITING.md](HANDWRITING.md) | Handwriting recognition: English + Urdu from day one, multi-model consensus, verification |
| [LANGUAGE_TRANSLATION.md](LANGUAGE_TRANSLATION.md) | Detect → original-language OCR → normalize → optional translate; originals preserved |
| [QUESTION_GENERATION.md](QUESTION_GENERATION.md) | Blueprint → scoped RAG → generation → validation → dedupe → rubric → review |
| [QUESTION_BANK.md](QUESTION_BANK.md) | Reusable question bank, versioning, semantic + lexical duplicate detection |
| [EXAM_ENGINE.md](EXAM_ENGINE.md) | Exam blueprints, exam types, canonical exam domain model, versioning |
| [EXAM_INTEGRITY.md](EXAM_INTEGRITY.md) | Integrity: unique IDs, QR/barcodes, randomization, locked publications, audit |
| [GRADING.md](GRADING.md) | Deterministic objective grading + rubric-based AI subjective grading |
| [TEACHER_VERIFICATION.md](TEACHER_VERIFICATION.md) | Review queue, teacher UI data contract, candidate-vs-final score separation |
| [PDF_DOCX.md](PDF_DOCX.md) | Template engine: canonical model → DOCX → PDF; branding, bilingual, answer sheets |
| [API.md](API.md) | Complete `/api/v1` surface: endpoints, authz, schemas, errors, pagination |
| [AUTH_RBAC.md](AUTH_RBAC.md) | Roles, permission matrix, tenant-isolation rules, JWT design |
| [SECURITY.md](SECURITY.md) | Security architecture incl. file validation, malware scanning, secrets, privacy |
| [STORAGE.md](STORAGE.md) | Object-storage abstraction, bucket layout, presigned URLs, dev driver |
| [QUEUES_WORKERS.md](QUEUES_WORKERS.md) | Celery queues, worker pools, idempotency, retries, progress, failure states |
| [SCALABILITY.md](SCALABILITY.md) | Horizontal-scaling path and service-extraction boundaries (post-MVP) |
| [OBSERVABILITY.md](OBSERVABILITY.md) | Logging, metrics, tracing, AI cost/quality dashboards |
| [TESTING.md](TESTING.md) | Test pyramid: unit, integration, API, E2E, security, perf, failure/recovery |
| [AI_EVALUATION.md](AI_EVALUATION.md) | **AI Evaluation Lab**: golden datasets, metrics, regression gates |
| [DEPLOYMENT.md](DEPLOYMENT.md) | Docker Compose deployment + Windows 11 / Docker Desktop / WSL2 requirements |
| [BACKUP_RECOVERY.md](BACKUP_RECOVERY.md) | Backup topology, RPO/RTO, restore drills |
| [COST_MANAGEMENT.md](COST_MANAGEMENT.md) | AI cost ledger, budgets, per-tenant reporting |
| [ROADMAP.md](ROADMAP.md) | Phases 0–11 with objectives, dependencies, deliverables, acceptance criteria, risks, tests |
| [RISKS.md](RISKS.md) | Risk register with mitigations |

**ADR index**: [ADR/README.md](ADR/README.md) — 16 records, see [ADR/](ADR/) for details.

## Core architectural principles (invariants)

1. **MongoDB is the system of record.** Qdrant is never the primary application database.
2. **Qdrant is the Derived Semantic Knowledge Index.** It holds rebuildable vector/search
   representations derived from MongoDB records + object-storage artifacts. It is not "just a
   cache" and not a record store; its content is always derivable from the sources.
3. **Object storage is the source artifact store.** Large files (books, papers, generated
   exams) never live in MongoDB documents or DBFS as primary storage.
4. **Frontend and backend are physically separated** (`frontend/` / `backend/`).
5. **AI output is NEVER automatically the final authority.** Applies uniformly to OCR,
   handwriting interpretation, translation, question generation, answer-key generation, and
   grading. AI results carry confidence + status fields; teacher verification is built in.
6. **Original student evidence is never deleted or overwritten.** Immutable, append-only.
7. **Teacher-approved scores are stored separately from AI candidate scores.**
8. **No provider lock-in.** LLMs, embeddings, and OCR all go through registries/adapters.
9. **MVP = modular monolith + Celery workers**, with clean domain boundaries so services can
   be extracted later if scale requires.
10. **Every AI artifact carries an AI Evidence Chain** — source materials, retrieved chunks,
    prompts, model info, and raw outputs that explain *why*.

## Reading order for implementers

1. PROJECT_OVERVIEW → REQUIREMENTS → ARCHITECTURE
2. DATABASE → MONGODB_SCHEMA → QDRANT → RAG → CHUNKING
3. AI_ARCHITECTURE → AI_PROVIDER_GATEWAY → MODEL_ROUTING → AI_EVALUATION
4. The pipeline docs for the feature area you are implementing
5. ADRs relevant to your area before writing code

## Status conventions

| Status | Meaning |
|---|---|
| **Approved** | Decision is settled; implement accordingly |
| **Proposed** | Recommended; awaiting review or affected by an open question |
| **Draft** | Work in progress, not authoritative |
| **Deferred** | Explicitly postponed to a later phase |

Every ADR records **Date**, **Status**, **Context**, **Decision**, **Consequences**, and
**Invalidation triggers**.

## Open questions

The running list of unresolved decisions is tracked in [ROADMAP.md](ROADMAP.md) and in each
relevant document. High-level open questions are summarized in
[PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md#open-questions).

## Change process

- Minor doc fixes: direct edit.
- Architectural changes: new ADR required (see [ADR/README.md](ADR/README.md)).
- This index must be updated whenever a document is added or removed.