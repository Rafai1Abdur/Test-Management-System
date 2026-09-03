# Architecture — AI-Powered School Test & Examination Management System

## 1. Architectural style

**Modular monolith + background workers** (ADR-0001). One deployable backend application
with strict domain boundaries, plus dedicated Celery worker processes. This is the
deliberate MVP choice: fastest path to production, simplest operations, and — because
domains are separated — ready to extract individual services if scale demands.
See [SCALABILITY.md](SCALABILITY.md).

Hard separations of concern:

| Data concern | Owner | Role |
|---|---|---|
| Structured application data | **MongoDB** | System of record |
| Vector / semantic search | **Qdrant** | Derived Semantic Knowledge Index (rebuildable) |
| Large artifacts (books, papers, PDF/DOCX) | **Object storage (S3/MinIO)** | Source artifact store |
| Job coordination | **Redis + Celery** | Queues, schedules, temporary state |
| AI models | **AI Gateway** (registry, router, adapters) | Gen, embeddings, OCR, translation |

## 2. C4 context

```
                   ┌─────────────────────────────────┐
                   │  Teacher / Principal / Admin /   │  (browser)
                   │  Exam Coordinator / Student      │
                   └───────────────┬─────────────────┘
                                   │ HTTPS
                         ┌─────────▼──────────┐
                         │  Frontend (Next.js) │───┐ REST /api/v1 JSON
                         └─────────┬──────────┘   │
                                   │              │
                         ┌─────────▼──────────┐    │
                         │  Backend (FastAPI) │◄───┘
                         │  + Celery workers  │
                         └───┬───────┬────┬───┘
                             │       │    │
                ┌────────────▼───┐ ┌─▼────┐ ┌───────────────────┐
                │ MongoDB (SoR)  │ │ Redis│ │ Qdrant (derived   │
                └────────────────┘ └──────┘ │  semantic index)  │
                                            └───────────────────┘
   Object storage (MinIO/S3) ◄── artifact store for uploads/generated files
   External AI providers (OpenAI/Gemini/Anthropic/GLM/Ollama/vLLM) ◄── AI Gateway
```

## 3. Domain boundaries (one codebase; strict import rules)

```
backend/app/domains/
  auth/          # identity, sessions, JWT
  school/        # schools, academic years
  enrollment/    # teachers, students, classes, sections, enrollments
  curriculum/    # subjects, chapters, topics, curriculum hierarchy, assessment periods,
                 # teaching/curriculum coverage (DRAFT→REVIEWED→LOCKED)
  materials/     # learning materials, ingestion pipeline, lifecycle, approval
  processing/    # OCR, handwriting, translation, page/region processing
  rag/           # chunking, indexing (Qdrant), retrieval, reranking (syllabus-aware scope)
  questions/     # question bank, generation, validation, deduplication
  exams/         # blueprints, scope resolution, examination sets, exam model, versions,
                 # paper instances, integrity
  grading/       # objective + AI subjective grading, verification queue
  results/       # approved results, aggregates, analytics
  ai/            # AI gateway, model registry, router, capabilities, evidence, cost
  audit/         # append-only audit, evidence snapshots
```

Rules: domains depend on `core/` + `infrastructure/` only; sibling-domain calls go through
domain-service interfaces; no cross-domain raw-collection access; no cross-domain object
imports beyond DTOs.

## 4. Containers

| Container | Purpose | MVP replicas | Horizontal path |
|---|---|---|---|
| `api` (FastAPI/uvicorn) | REST `/api/v1`, auth, orchestration | 1 | Scale statelessly behind LB |
| `worker-ingestion` | Material extraction, analysis, chunking | 1 | Add workers |
| `worker-embeddings` | Embedding + Qdrant indexing | 1 | Add workers (GPU optional) |
| `worker-generation` | Question/rubric generation | 1 | Add workers |
| `worker-ocr` | Page/region/OCR/handwriting/translation | 1 | Add workers |
| `worker-grading` | AI grading evals | 1 | Add workers |
| `worker-export` | PDF/DOCX generation, analytics jobs | 1 | Add workers |
| `mongo` | Primary DB | 1 | Replica set later |
| `redis` | Broker, cache | 1 | Sentinel later |
| `qdrant` | Derived index | 1 | Distributed later |
| `minio` | S3-compatible storage | 1 | Cloud S3 later |

## 5. Planned repository tree

```
frontend/                    Next.js + React + TS (no backend logic)
  src/app/                   App Router pages/routes
  src/components/            UI primitives
  src/features/              feature modules (materials, questions, exams, grading, ...)
  src/lib/                   API client, auth context, permissions
  src/types/                 shared TS types (mirrored from OpenAPI schema)
backend/                     FastAPI modular monolith + Celery workers
  app/
    api/v1/                  routers (one per domain)
    core/                    config, security, logging, tenancy context
    domains/                 (see §3)
    infrastructure/          db (pymongo), storage, queue, search, ai_gateway, observability
  workers/                   celery app, task modules per worker pool
  tests/                     unit + integration
infrastructure/              docker-compose.yml, env templates, nginx, backup scripts
 docs/                       this specification
 scripts/                    dev/ops helper scripts
 tests/                      cross-app e2e/contract tests
 evals/                      AI Evaluation Lab (golden datasets, config, results)
```

## 6. Key architectural invariants (enforced in code review and tests)

1. MongoDB is the only database accessed by domain repositories; Qdrant only via `rag` domain.
2. All tenant-scoped queries MUST include `school_id` (middleware-injected; see
   [AUTH_RBAC.md](AUTH_RBAC.md)).
3. Every AI side effect writes evidence + `model_runs` + `ai_usage` entries (see
   [AI_ARCHITECTURE.md](AI_ARCHITECTURE.md) and [COST_MANAGEMENT.md](COST_MANAGEMENT.md)).
4. AI outputs have status ⊆ {`CANDIDATE`, `NEEDS_REVIEW`, `APPROVED`, `REJECTED`} and never
   mutate a final score directly ([AI_ARCHITECTURE.md](AI_ARCHITECTURE.md)).
5. Long tasks are never executed inside an HTTP request handler — always queued
   ([QUEUES_WORKERS.md](QUEUES_WORKERS.md)).
6. Files always go through the storage abstraction — never through the DB
   ([STORAGE.md](STORAGE.md)).
7. Frontend never touches MongoDB/Qdrant/storage directly.
8. AI model and embedding model selections are registry-driven — no hard-coded providers
   ([MODEL_ROUTING.md](MODEL_ROUTING.md), [QDRANT.md](QDRANT.md)).
9. Qdrant content is always derivable from MongoDB + object storage (rebuild contract).
10. Published exams are immutable; corrections require a new version
    ([EXAM_INTEGRITY.md](EXAM_INTEGRITY.md)).
11. **Generation scope is syllabus-governed** (ADR-0017): `generation_scope ⊆
    approved_teaching_coverage ∩ blueprint_scope`; retrieval is filter-time scoped, never
    by index mutation; Qdrant payloads remain period-independent.
12. Assessment Period, Exam Type, Examination Set, and Exam are **distinct concepts** and
    are never merged (ADR-0017).

## 7. Decisions & trade-offs summary

| Decision | Documented in |
|---|---|
| Modular monolith, no microservices | ADR-0001 |
| FastAPI + Next.js physical split | ADR-0002 |
| MongoDB system of record | ADR-0003, MONGODB_SCHEMA.md |
| PyMongo Async API (no Motor) | ADR-0006, DATABASE.md |
| Qdrant = Derived Semantic Knowledge Index | ADR-0007, QDRANT.md |
| Embedding registry (bge-m3 default) | ADR-0008, MODEL_ROUTING.md |
| AI gateway (no provider lock-in) | ADR-0009, AI_PROVIDER_GATEWAY.md |
| OCR gateway (no engine lock-in) | ADR-0010, OCR.md |
| Canonical exam model → PDF/DOCX | ADR-0011, PDF_DOCX.md |
| AI never final authority | ADR-0012, AI_ARCHITECTURE.md |
| Docker Compose + Windows/WSL2 dev | ADR-0013, DEPLOYMENT.md |
| AI evidence chain | ADR-0014 |
| Exam integrity | ADR-0015, EXAM_INTEGRITY.md |
| AI Evaluation Lab | ADR-0016, AI_EVALUATION.md |
| Assessment periods, teaching coverage, exam sets, syllabus-aware scope | ADR-0017 |

See [ADR/README.md](ADR/README.md) for the full decision log.