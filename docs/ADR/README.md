# Architecture Decision Records (ADR)

Each ADR records: **Date · Status · Context · Decision · Consequences (positive/negative) ·
Invalidation triggers.** Status values: Approved / Proposed / Draft / Superseded.

| ADR | Title | Status |
|---|---|---|
| [ADR-0001](ADR-0001-modular-monolith-over-microservices.md) | Modular monolith + workers over microservices | Approved |
| [ADR-0002](ADR-0002-fastapi-nextjs-split.md) | FastAPI backend / Next.js frontend, physically separate | Approved |
| [ADR-0003](ADR-0003-mongodb-primary-jwt-rbac-tenancy.md) | MongoDB as system of record with JWT+RBAC+tenant isolation | Approved |
| [ADR-0004](ADR-0004-celery-redis-queue.md) | Celery + Redis for background work | Approved |
| [ADR-0005](ADR-0005-minio-s3-storage-abstraction.md) | S3-compatible object storage abstraction | Approved |
| [ADR-0006](ADR-0006-pymongo-asyncclient-data-access.md) | PyMongo Async API (`AsyncMongoClient`), no Motor | Approved |
| [ADR-0007](ADR-0007-qdrant-derived-semantic-knowledge-index.md) | Qdrant as Derived Semantic Knowledge Index | Approved |
| [ADR-0008](ADR-0008-embedding-model-registry.md) | Embedding model registry; bge-m3 MVP default | Approved |
| [ADR-0009](ADR-0009-ai-gateway-provider-adapters.md) | AI gateway with provider adapters (no lock-in) | Approved |
| [ADR-0010](ADR-0010-ocr-provider-abstraction.md) | OCR/HTR provider abstraction | Approved |
| [ADR-0011](ADR-0011-canonical-exam-model-pdf-docx.md) | Canonical exam model → template → PDF/DOCX | Approved |
| [ADR-0012](ADR-0012-ai-output-never-final-authority.md) | AI output is never the final authority | Approved |
| [ADR-0013](ADR-0013-docker-compose-windows-wsl2-dev.md) | Docker Compose single-server + Windows/WSL2 dev rules | Approved |
| [ADR-0014](ADR-0014-ai-evidence-chain.md) | Mandatory AI Evidence Chain per artifact | Approved |
| [ADR-0015](ADR-0015-exam-integrity.md) | Exam integrity as first-class domain | Approved |
| [ADR-0016](ADR-0016-ai-evaluation-lab.md) | AI Evaluation Lab with golden datasets + gates | Approved |
| [ADR-0017](ADR-0017-assessment-periods-coverage-exam-sets.md) | Assessment periods, teaching coverage, examination sets, syllabus-aware scope | Approved |

**Process:** propose → review → approve; architectural changes require a new ADR. Superseding
an ADR requires a new ADR referencing it.