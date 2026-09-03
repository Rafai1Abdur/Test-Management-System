# Database Design — MongoDB

## 1. Role

MongoDB is the **system of record** for all structured application data. It is *not* a
blob store (large files → object storage, [STORAGE.md](STORAGE.md)) and *not* the vector
index (Qdrant is the derived semantic knowledge index, [QDRANT.md](QDRANT.md)). Anything
that needs transactional consistency, queryability, or auditing lives in MongoDB.

## 2. Data-access layer

- **Driver: PyMongo Async API** — `pymongo.AsyncMongoClient`, PyMongo ≥ 4.9 (target
  4.17.x). No Motor, no ORM/ODM (see ADR-0006). We deliberately use Pydantic v2 models
  for validation and a **repository layer** per domain.
- Repositories own all collection access for their domain; the `db` package provides
  client lifecycle, connection pooling, and typed accessors. Transactions are used only
  where multi-document atomicity is required (e.g., publishing an exam version).
- Async + FastAPI: one shared client; ensure indexes are created at startup (idempotent).
- Sync contexts (Celery workers) use the same AsyncMongoClient inside async task bodies.

## 3. Modeling philosophy

We **do not normalize MongoDB like a relational database**. Decisions are deliberate:

| Pattern | Where used | Why |
|---|---|---|
| **Reference by ID** | cross-entity relations (question → subject, chapter, material) | Avoids unbounded documents; enables independent lifecycle & versioning |
| **Embed (small, owned, bounded)** | question options, rubric criteria, evidence headers, answer snapshots | Read paths get one round-trip; always displayed together |
| **Embed with cap** | recent history arrays (e.g., last 5 grading attempts) | Bounded growth |
| **Separate collection** | large/append-heavy data: audit_logs, ocr_results, model_runs, ai_usage, student_answers | Query patterns differ; TTL/retention applies per collection |
| **Denormalized rollups** | class_result / subject_result / chapter_performance | Precomputed aggregates for analytics reads; rebuilt by workers |
| **Embedded, bounded (coverage)** | `curriculum_coverage.chapters[]` (chapter statuses per period+subject) | Chapters/topics per subject-period are small, always read with the coverage record (ADR-0017) |
| **Referenced (assessment)** | `assessment_periods`, `examination_sets` referenced by blueprints/exams/results | Shared across subjects/exams; own lifecycle and publication state (ADR-0017) |
| **Immutable append-only** | audit_logs, evidence snapshots, original answers, ocr raw outputs | Auditability & evidence chain |

General rules:
- IDs: `_id` = ObjectId for internal refs; **public ULID strings** for anything a user or
  machine-readable surface sees (exam IDs, paper IDs, question public IDs) — QR/URL-safe,
  sortable. Enforce with app-layer validation.
- Every tenant-scoped document carries `school_id`; every school-scoped document carries
  `academic_year_id` where applicable. Assessment/reporting collections also carry
  `assessment_period_id` (and `examination_set_id` where relevant) so period is a first-class
  query/report dimension (FR-63).
- Timestamps: `created_at`, `updated_at` (UTC). Evidence/history records add `recorded_at`.
- Soft delete only where business meaning requires (questions reference history); hard
  delete only in compliance/deletion workflows (student personal-data deletion, backup-restore).

## 4. Tenancy (multi-school)

- Super Admin operates across schools; every other principal is school-scoped.
- Tenant isolation is **enforced at the repository layer** (mandatory `school_id` filter
  injected from the request context), never left to callers.
- Indexes always include `school_id` as the leading key for tenant-scoped collections.
- Cross-school access attempts → Authorization error; audit them.

## 5. Naming & conventions

- Collections: `snake_case`, plural nouns: `users`, `schools`, `exam_paper_instances`.
- No collection-level fixed schema by default; implement **MongoDB JSON Schema
  `$jsonSchema` validation** on critical collections (users, schools, questions, exams,
  results) in addition to Pydantic request validation.
- Indexes named `idx_<fields>_<dir>` for maintainability.

## 6. Index & connection strategy

- Every queryable field that appears in a filter/sort gets an index (see
  [MONGODB_SCHEMA.md](MONGODB_SCHEMA.md) per collection). Leading `school_id` first.
- TTL indexes for transient data (signed-url records, verification tokens).
- Connection pool sized by config; `maxPoolSize` guidance 50 per process; monitored via
  observability (see [OBSERVABILITY.md](OBSERVABILITY.md)).

## 7. Retention & lifecycle

| Data class | Retention |
|---|---|
| Assessment periods & coverage records | Retain with the academic year + retention policy; coverage history (incl. locked snapshots) archived with the year |
| Audit logs | Retain for platform lifetime; archive after 3+ years |
| Student papers & evidence | At minimum until end of academic year + defined retention (default 2 years); never auto-deleted without policy |
| OCR/translation intermediate | Keep until grading verified; raw evidence retained per above |
| AI usage cost ledger | Aggregated monthly summaries kept indefinitely; raw rows archive after 1 year |
| Jobs/processing logs | 30 days (with error samples kept) |
| Generated exams (published versions) | Retain all versions; only ARCHIVED state |
| Session/temporary | TTL |

See [SECURITY.md](SECURITY.md) for GDPR-style deletion workflows and
[BACKUP_RECOVERY.md](BACKUP_RECOVERY.md) for backup topology.

## 8. Migrations

- Schema evolution is **forward-compatible**: additive fields always; field renames/removals
  done via release scripts with backfill jobs, never by breaking existing readers.
- Migration runner: versioned scripts in `backend/app/infrastructure/db/migrations/`,
  recorded in `schema_migrations`. Migrations are idempotent.

## 9. Relationship to Qdrant and object storage

- Qdrant points reference MongoDB `_id`s (`chunk_id`, `material_id`, `question_id`,
  `document_id`). Qdrant is **derived** and rebuildable: chunks and their metadata live in
  MongoDB (`rag_chunks`, `learning_materials`); vectors live in Qdrant.
- Object storage keys are stored in MongoDB fields (`storage_key` / `file_ref`).

## 10. Transactions & concurrency

- Multi-document transactions: publish exam version; finalize attempt; approve material +
  flag dependent questions. Kept rare to avoid hot contention.
- Optimistic concurrency: `rev` (revision) integer on versioned docs (questions, exam
  versions, blueprints, rubrics).
- Unique constraints via unique indexes (e.g., `(school_id, code)` for subjects).