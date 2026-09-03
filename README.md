# AI-Powered School Test & Examination Management System

> **Status: Phase 0 — Documentation & Architecture.**
> The repository currently contains **documentation only**. No application code has been written.

## What this project is

A scalable educational assessment platform that lets schools manage teachers, students,
classes, curriculum, and learning materials; ingest books in many formats; build a
curriculum-aware RAG knowledge index; generate examinations from exam blueprints;
print PDF/DOCX exams with machine-readable paper identifiers; process scanned student
papers (including English and Urdu handwriting); grade objectively and with AI
assistance; route uncertain results to teacher verification; and provide analytics —
all with full auditability and no vendor lock-in for AI models.

## Repository layout (planned)

| Path | Responsibility |
|---|---|
| `frontend/` | Next.js + React + TypeScript application. Physically separated from the backend. Not yet created. |
| `backend/` | Python 3.11+ / FastAPI application, modular-monolith domains, Celery workers. Not yet created. |
| `infrastructure/` | Docker Compose topology, env templates, Nginx/TLS, backup scripts. Not yet created. |
| `docs/` | **The authoritative specification.** Created in this phase. Start here. |
| `scripts/` | Developer/ops helper scripts (env checks, DB tooling). Not yet created. |
| `tests/` | Cross-cutting E2E/contract test suites. Unit/integration tests live in each app. Not yet created. |
| `evals/` | AI Evaluation Lab: golden datasets, eval harness configs, eval results. Not yet created. |

## Documentation

Read **[docs/README.md](docs/README.md)** first — it is the index and entry point.

## Core architectural principles (invariants)

1. **MongoDB is the system of record** (PyMongo `AsyncMongoClient`; no Motor).
2. **Qdrant is the Derived Semantic Knowledge Index** — rebuildable from MongoDB + object storage.
3. **Object storage is the source artifact store** (S3-compatible; MinIO in dev).
4. **Frontend and backend are physically separated**.
5. **AI output is NEVER automatically the final authority.**
6. **Original student evidence is never deleted or overwritten**.
7. **Teacher approval is authoritative** for AI-assisted grading; teacher-approved and AI-candidate scores are stored separately.
8. **No one-provider lock-in** for LLMs, embeddings, or OCR — everything goes through gateways/registries.
9. **MVP = modular monolith + background workers**; domain boundaries prepared for later service extraction.
10. **Every AI artifact carries an evidence chain** and is auditable.

See [docs/ADR/](docs/ADR/) for the rationale behind each decision.

## Reference stack (recorded in ADRs)

| Layer | Choice |
|---|---|
| Frontend | Next.js (App Router) + React + TypeScript |
| Backend | Python 3.11+ · FastAPI · Pydantic v2 |
| App DB | MongoDB 7 · PyMongo Async API (`AsyncMongoClient`) |
| Vector/search | Qdrant (derived semantic knowledge index; dense + sparse + hybrid + rerank) |
| Embeddings | Registry-driven; **bge-m3 is the MVP default** (configurable, re-indexable) |
| AI providers | Adapter gateway — OpenAI, Gemini, Anthropic, GLM, Ollama, vLLM, others |
| OCR/handwriting | OCR provider abstraction (Tesseract + PaddleOCR + VLM adapters); English & Urdu |
| Queue | Celery + Redis, separate queues per worker pool |
| Object storage | S3 abstraction (MinIO in dev; any S3 in prod) |
| Deployment | Single-server Docker Compose (MVP); Windows 11 + Docker Desktop + WSL2 as dev environment |

## Milestones

See [docs/ROADMAP.md](docs/ROADMAP.md). Phases 0–11 from foundation to production hardening.