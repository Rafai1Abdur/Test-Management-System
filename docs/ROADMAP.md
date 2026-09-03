# Implementation Roadmap

Phases below are sequential with soft overlaps. Each phase lists Objectives, Dependencies,
Deliverables, Acceptance criteria, Risks, Tests.

## Phase 0 — Documentation & architecture ✅ (current)
- **Objectives:** authoritative spec; ADR log; scope decisions.
- **Deliverables:** this `docs/` package (see README index) + ADRs 0001–0016.
- **Acceptance:** every planned doc exists; invariants checkable; repo foundation
  (.gitignore/.gitattributes); no application code.
- **Exit gate:** docs review sign-off; open questions resolved for Phase 1 items.

## Phase 1 — Foundation
- **Objectives:** backend app factory (FastAPI + Pydantic v2, config, logging) with
  PyMongo Async (`AsyncMongoClient`) data access; frontend scaffold (Next.js App Router,
  TS, API client, auth context); infra compose for mongo/redis/qdrant/minio + nginx;
  health endpoints; CI skeleton; `scripts/check_env`.
- **Dependencies:** Phase 0; Windows/WSL2 tooling verified.
- **Deliverables:** `backend/` skeleton + health; `frontend/` login shell; `infrastructure/
  docker-compose.yml`; CI (lint, unit, build).
- **Acceptance:** `docker compose up` runs infra + API on dev; `/healthz` & `/readyz`;
  migrations track state; frontend talks to API; `.env.example` only.
- **Risks:** driver/async quirks; WSL2 volume performance — mitigate per DEPLOYMENT §2.
- **Tests:** unit (config/logging/health), integration (Mongo connect), smoke E2E.

## Phase 2 — Users, tenancy, curriculum, materials
- **Objectives:** schools/years/classes/sections/subjects/chapters/topics; auth (JWT
  refresh, RBAC, tenancy middleware); users/teachers/students/enrollments; material
  upload+validation+storage; ingestion pipeline (extract/language/structure/curriculum
  mapping candidate) with lifecycle + approval; **assessment periods; teaching coverage
  (chapter-level mandatory, section/topic optional; DRAFT→REVIEWED→LOCKED; class/section
  override with grade+subject default inheritance); examination sets** ([ADR-0017](ADR/ADR-0017-assessment-periods-coverage-exam-sets.md)).
- **Dependencies:** Phase 1.
- **Deliverables:** domain services + repositories + API groups (auth, users, schools,
  teachers, students, classes, curriculum, materials, **assessment-periods,
  curriculum-coverage, examination-sets**); ClamAV integration; jobs skeleton.
- **Acceptance:** RBAC+tenancy tests pass; multi-format extraction (MVP formats);
  material reaches NEEDS_REVIEW/APPROVED via worker; uploads stored in MinIO.
- **Risks:** DOC/PPT legacy conversion; malware-scan availability.
- **Tests:** authz matrix, tenancy isolation, format-adapter fixtures, lifecycle tests.

## Phase 3 — RAG
- **Objectives:** hierarchical chunking; `rag_chunks`; embedding registry (bge-m3
  default) + Qdrant collections (`chunks_live` alias); dense+sparse+RRF+rerank; scoped
  retrieval API; rebuild job.
- **Dependencies:** Phase 2 (approved materials).
- **Deliverables:** chunking + indexing + retrieval services; `/knowledge/*`; `rag` eval
  dataset + gates.
- **Acceptance:** retrieval hit@k gates green; allowlist/authority enforced; re-index
  alias-swap demo; Qdrant tenancy filter verified.
- **Risks:** embedding quality for Urdu; payload-index tuning.
- **Tests:** RAG eval, rebuild drill, filter correctness, perf smoke.

## Phase 4 — Question bank & generation
- **Objectives:** question/version/rubric models; duplicate detection (lexical+semantic);
  generation pipeline (blueprint cells → scoped RAG → generate → validate → dedupe →
  rubric → review); teacher approval UI; **scope resolution from approved coverage**
  (modes; materialization at blueprint creation + generation-time revalidation, fail-closed,
  never silently expanded); **weighting-tolerance validation** (±10 pp default, configurable).
- **Dependencies:** Phase 3.
- **Deliverables:** `/questions*`, `/exam-blueprints` (+ `/resolve-scope`), generation
  workers; QG eval set.
- **Acceptance:** correctness ≥ 90% (sampled), duplicate rate ≤ 5%, no auto-approval;
  evidence chain on every candidate; out-of-scope generation fails closed (0 violations);
  weighting adherence gate.
- **Risks:** LLM hallucinated keys; schema drift.
- **Tests:** QG eval gates, validation unit tests, dedupe precision/recall.

## Phase 5 — Exam builder & PDF/DOCX
- **Objectives:** canonical exam model; validation + publish lock; paper instances with
  QR/barcode + randomization; PDF (WeasyPrint); DOCX (Phase 2 lane); bilingual layout;
  answer-key RBAC + audit; **examination-set publication; syllabus lock** (scope immutable
  at APPROVED/PUBLISHED — post-publication coverage change ⇒ new version).
- **Dependencies:** Phase 4.
- **Deliverables:** `/exams` + `/papers` + export workers; artifacts in MinIO;
  `/examination-sets/*` (publish).
- **Acceptance:** publish-immutability tests pass; QR round-trip decodes; RTL Urdu PDF
  renders and matches region templates; key access audited; syllabus-lock + new-version
  flow tested.
- **Risks:** RTL/PDF font issues; randomization reproducibility.
- **Tests:** layout golden fixtures, integrity tests, render smoke.

## Phase 6 — Student paper processing
- **Objectives:** submission upload (multi-page); page/region detection; perspective/
  rotation/crop/enhance; printed OCR (EN/UR: Tesseract/PaddleOCR) + machine-readable
  papers; answer reconstruction + mapping; low-confidence → repair UI.
- **Dependencies:** Phase 5.
- **Deliverables:** `/submissions`, `/ocr/*`, worker-ocr; OCR eval sets.
- **Acceptance:** OCR CER/WER gates; page/region success rate; QR multi-page mapping;
  originals immutable.
- **Risks:** photo quality variance; Urdu printed fonts.
- **Tests:** OCR eval, region fixtures, crash-recovery tests.

## Phase 7 — Handwriting & language
- **Objectives:** handwriting multi-model consensus (EN lane MVP; **UR same pipeline
  architecturally**); language detection + translation (optional); Urdu normalization/RTL;
  strict confidence gates + verification surfacing.
- **Dependencies:** Phase 6.
- **Deliverables:** HTR adapter pack; consensus service; translation service; HW eval
  sets (EN + UR).
- **Acceptance:** EN HW CER gates; UR HW pipeline exists with eval set + strict gates;
  originals preserved; translations never replace originals.
- **Risks:** UR handwriting quality (highest); consensus disagreement handling.
- **Tests:** HW eval (EN/UR), translation eval, RTL/diacritic fixtures.

## Phase 8 — AI grading
- **Objectives:** objective deterministic grading; AI rubric grading (candidate +
  confidence + evidence); verification queue; results computation (student/class/subject/
  chapter — **all keyed by academic year + assessment period**); basic analytics cubes;
  grading eval set.
- **Dependencies:** Phases 6–7.
- **Deliverables:** `/grading`, `/verifications`, `/results`, worker-grading, basic
  `/analytics`.
- **Acceptance:** MAE/κ gates; candidate-vs-final enforced; override-rate telemetry live;
  no auto-finalization below thresholds.
- **Risks:** grading inconsistency; borderline answers.
- **Tests:** grading eval, verification E2E, results idempotency.

## Phase 9 — Teacher verification & results polish
- **Objectives:** verification workspace UX (bundle view, bulk-accept, edit, audit),
  result reports (**report cards per assessment period + examination set** — school-branded
  templates + platform default), notifications, dashboard polish.
- **Dependencies:** Phase 8.
- **Acceptance:** usability passes; verify-time targets; audit completeness.
- **Tests:** E2E verification journeys; permission tests.

## Phase 10 — Analytics & reporting
- **Objectives:** advanced analytics (mastery, discrimination, trends), report templates,
  export.
- **Dependencies:** Phase 9.
- **Acceptance:** dashboards match spec; rollups fresh; exports RBAC.
- **Tests:** analytics accuracy fixtures.

## Phase 11 — Production hardening & scaling
- **Objectives:** PITR/replica-set decision, load tests, security review, backup drills,
  cost billing import, worker scale-out validation, prod runbooks.
- **Dependencies:** Phases 1–10.
- **Acceptance:** load/soak targets; restore drill Green; security scan pass; runbooks
  current.
- **Tests:** perf/load, failure drills, DR drill.

## MVP slice (Phases 1–6 + basic 8) recap
See [PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md). Deliberately **not** in MVP: DOCX render,
UR handwriting delivery (architecture in place), translation default-on, matching/ordering/
numerical types, advanced analytics. **In MVP:** assessment periods, teaching coverage,
examination sets, scope modes, syllabus lock ([ADR-0017](ADR/ADR-0017-assessment-periods-coverage-exam-sets.md)).

## Roadmap assumptions & open questions
Tracked in PROJECT_OVERVIEW §Open questions. Phases list the decisions they need resolved
(e.g., Phase 4 provider defaults; Phase 5 per-student paper printing scope).