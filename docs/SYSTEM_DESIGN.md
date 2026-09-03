# System Design — Workflows & Flows

Sequence/flow designs for the major workflows. Notation: `User → Frontend → API → Domain
→ Infrastructure → (Worker/External)`. All entry points are REST `/api/v1` (see
[API.md](API.md)). Long-running work is always enqueued
([QUEUES_WORKERS.md](QUEUES_WORKERS.md)).

## 0. Global cross-cutting behavior

- **Tenancy**: every request resolves `school_id` from the authenticated principal
  (JWT claim or role binding); domain queries always include it (`AUTH_RBAC.md`).
- **Idempotency**: mutating endpoints accept an `Idempotency-Key` header; duplicate keys
  return the original result (`API.md`).
- **Audit**: every mutation writes an `audit_logs` entry with actor, action, before/after,
  timestamp, tenant (`SECURITY.md`).

## 1. Curriculum setup

```
Teacher/Admin → POST /api/v1/academic-years  → create year (school-scoped)
Teacher/Admin → POST /api/v1/classes & /sections
Teacher/Admin → POST /api/v1/subjects (grade, language)
Teacher/Admin → POST /api/v1/chapters  (subject_id, order, objectives)
Teacher/Admin → POST /api/v1/topics    (chapter_id)
Student       → POST /api/v1/enrollments (bulk CSV or per-student)
Teacher/Admin → POST /api/v1/assessment-periods (academic_year_id, name, type, order, dates)
Teacher       → POST/PATCH /api/v1/curriculum-coverage (period, subject, optional class/section,
               chapter statuses; chapter-level mandatory, topic-level optional)
Teacher/Admin → POST /api/v1/curriculum-coverage/{id}/lock (DRAFT → REVIEWED → LOCKED)
```
Results: curriculum tree fully stored in MongoDB collections
(`academic_years`, `assessment_periods`, `curriculum_coverage`, `classes`, `sections`,
`subjects`, `chapters`, `topics`, `enrollments`). No jobs required.

**Coverage inheritance:** resolved coverage for a class/section is a **per-chapter merge**
of the class/section-specific record and the grade+subject default (override status wins
where both exist; default status applies where the override omits the chapter — a partial
override never implies that unlisted chapters do not exist) (ADR-0017). Blueprint scope
resolution follows this rule and records per-chapter provenance (`default` |
`class_override`).

## 2. Material ingestion & approval

```
Teacher → POST /api/v1/materials (multipart file + metadata)
  API validates size/type → sec-validates → malware-scan (async) → stores to
  object storage `materials/` → creates material doc (status=UPLOADED) →
  enqueue job `material.process`
Worker-ingestion:
  validate & format-detect → content extract (format adapter)
  → OCR/vision if image/scan → normalize → detect language
  → detect subject/grade/chapter/section/topic (AI, candidate only)
  → hierarchical chunking → store chunks (MongoDB) → status=ANALYZED|NEEDS_REVIEW
Teacher → GET /api/v1/materials/{id}/analysis → reviews AI metadata → corrects
Worker-embeddings: embed chunks (bge-m3 default) → index into Qdrant collection
  (payload: material/subject/chapter/section/topic/grade/language/authority…)
Teacher → PATCH /api/v1/materials/{id}/approve → status=APPROVED
  (only APPROVED materials are retrievable by the exam generator)
```
States: `UPLOADED → PROCESSING → ANALYZED → NEEDS_REVIEW → APPROVED → ARCHIVED`
(and `FAILED` from any point). Detail: [MATERIAL_INGESTION.md](MATERIAL_INGESTION.md).

## 3. RAG retrieval (internal service, not a user endpoint)

```
Caller (question generator / grading) → rag.retrieve(query, filter, k)
  build filter: school_id, material_ids[], subject/chapter/section, grade, language,
                authority, content_type, approved_only=True
  dense search (embedding model registered) + sparse search (BM25/SPLADE-style)
  → hybrid fusion (RRF) → rerank (cross-encoder or LLM) → top-k chunks
  → context assembly (dedupe, respect token budget, order by provenance)
  → return ChunkRef[] with payloads (material_id, chapter, section, page, text, score)
```
Detail: [RAG.md](RAG.md), [QDRANT.md](QDRANT.md).

## 4. Question generation & review

```
Teacher → POST /api/v1/exam-blueprints  (school, year, **assessment_period**, **exam set**,
   grade, subject, materials[], **scope mode**, chapters/sections, **weighting + tolerance**,
   total_marks, duration, question counts/types, difficulty dist.,
   Bloom dist., language, instructions)
   → POST /api/v1/exam-blueprints/{id}/resolve-scope → scope materialized from approved
     **locked coverage** (teacher reviews while DRAFT; blueprint `rev` recorded on every
     generation run — EXAM_ENGINE §2)
Worker-generation:
  → **revalidate materialized scope vs applicable coverage; fail closed if invalid**
  for each (chapter, type, difficulty) cell:
    **syllabus-aware** scoped RAG retrieval (blueprint scope ∩ coverage ∩ materials only)
    → candidate generation (LLM, provider via gateway, structured assessment context)
    → structural validation (JSON schema per type)
    → answer key validation (MCQ must have exactly one correct, etc.)
    → curriculum alignment check → difficulty check → **weighting-tolerance check**
    → duplicate detection (semantic + lexical against bank)
    → rubric generation → quality pass (another model or self-check)
  questions stored w/ status=CANDIDATE + evidence (source chunks, prompt version, model,
  **scope provenance**)
Teacher → GET /api/v1/questions?status=CANDIDATE → approve/edit/reject per question
  Approved questions can be added to exam drafts.
```
Detail: [QUESTION_GENERATION.md](QUESTION_GENERATION.md),
[QUESTION_BANK.md](QUESTION_BANK.md).

## 5. Exam creation, publishing & printing

```
Teacher/ExamCoord → POST /api/v1/examination-sets (period, grade, class/section scope,
  subjects, advisory schedule) → set status DRAFT → SCHEDULED
Teacher → POST /api/v1/exams (draft from blueprint + bank selections; linked to exam set
  + assessment period; **syllabus frozen on READY**)
  → exam version v1 frozen on publish; answer key stored separately, RBAC-gated
Exam Coordinator → POST /api/v1/exams/{id}/publish
  → lock exam (no edits; syllabus/scope immutable) → optionally generate randomized paper
    instances (POST /api/v1/exams/{id}/papers) each with unique paper_id + QR + shuffled order
Worker-export: canonical exam model → PDF (WeasyPrint) / DOCX (python-docx/docxtpl)
  → stored in object storage `exams/` → download via signed URL
Teacher → GET /api/v1/exams/{id}/key → requires exam.key:read permission; audited
```
Detail: [EXAM_ENGINE.md](EXAM_ENGINE.md), [EXAM_INTEGRITY.md](EXAM_INTEGRITY.md),
[PDF_DOCX.md](PDF_DOCX.md).

## 6. Student paper upload & processing

```
Teacher → POST /api/v1/submissions (multipart: exam_id/paper_id + images/pdfs,
   student ref) → store originals to `papers/` → create attempt (status=UPLOADED) →
   enqueue `paper.process`
Worker-ocr:
  page detection → perspective/rotation correction → crop → enhance
  → question/answer region detection (template-aware via paper_id/QR metadata)
  → OCR per region (printed: Tesseract/PaddleOCR; handwritten: multi-model consensus)
  → language detection → reconstruct answers → map to question_ids
  → produce OCR results + confidence + evidence (crops, raw outputs)
  → optional translation (store separately, keep original)
Teacher: missing/questionable regions flagged (confidence < threshold) → fix in review UI
```
Detail: [OCR.md](OCR.md), [HANDWRITING.md](HANDWRITING.md),
[LANGUAGE_TRANSLATION.md](LANGUAGE_TRANSLATION.md).

## 7. Grading & teacher verification

```
Worker-grading (per answer):
  objective questions (MCQ/TF/…) → deterministic compare vs approved answer key → score
  subjective (short/long/essay) → AI rubric grading:
     input = question + official answer + rubric + student answer (original + translation)
             + relevant curriculum context (RAG) + evidence
     output = criterion marks + total + explanation + confidence (status=CANDIDATE)
  write grading_results with full evidence chain; final score not set
System policy: confidence < threshold (per question type / language) → status=NEEDS_REVIEW
Teacher → GET /api/v1/verifications → review queue (original paper, crop, OCR,
  translation, question, key, rubric, AI score, confidence, explanation)
Teacher → accept (→ APPROVED) / modify score / correct OCR-answer / reject
  final graded score = teacher-approved value (separate field); graders audit trail updated
System policy: the answer-key approver of an exam must not be its sole final verifier
(AUTH_RBAC §2b).
Results engine: approved scores → student_attempt summaries → student/class/subject/
  chapter analytics
```
Detail: [GRADING.md](GRADING.md), [TEACHER_VERIFICATION.md](TEACHER_VERIFICATION.md),
[OBSERVABILITY.md](OBSERVABILITY.md).

## 8. Results & analytics

```
On finalization of an attempt (all answers verified):
  results service computes student_result, rolls into class_result, subject_result,
  chapter_performance, question stats (correctness, discrimination), analytics docs
  (precomputed or on-demand per READ path)
Grade computation: `student_results.grade` = percentage → applicable `grading_scales`
entry (subject-specific > grade-level > school default) → matching band → grade/pass-fail.
No applicable scale ⇒ `grade = null` (never guessed) + warning. Detail: MONGODB_SCHEMA
`grading_scales`.
Admins/principals → GET /api/v1/analytics/... (paginated, filtered by tenant)
```

## 9. State machines (canonical)

| Entity | States |
|---|---|
| Assessment period | `PLANNED → ACTIVE → CLOSED` |
| Curriculum coverage | `DRAFT → REVIEWED → LOCKED` (changes after lock require new exam version) |
| Examination set | `DRAFT → SCHEDULED → PUBLISHED → COMPLETED → ARCHIVED` |
| Learning material | `UPLOADED → PROCESSING → ANALYZED → NEEDS_REVIEW → APPROVED → ARCHIVED; FAILED` |
| Question | `CANDIDATE → NEEDS_REVIEW → APPROVED → DEPRECATED; REJECTED; (versioned)` |
| Exam | `DRAFT → READY (syllabus frozen) → PUBLISHED (locked) → ARCHIVED` |
| Paper instance | `GENERATED → PRINTED → RETURNED → PROCESSING → GRADED → VERIFIED` |
| Grading result | `PENDING → CANDIDATE → NEEDS_REVIEW → APPROVED/REJECTED` |
| Job | `QUEUED → RUNNING → SUCCEEDED | FAILED(RETRYABLE/NON-RETRYABLE) → CANCELED` |

Transitions are validated in the domain layer; every transition emits an audit event.

## 10. Failure & retry policy (summary)

- Jobs are idempotent (keyed by domain id + job type); retries use exponential backoff with
  max 5 attempts by default; poison tasks → dead-letter queue + alert.
- Worker crashes mid-job → state remains `RUNNING`; a watchdog requeues stale tasks
  (details in [QUEUES_WORKERS.md](QUEUES_WORKERS.md)).
- AI provider failure → router tries next provider in the task's eligible set, then fails
  the job as `FAILED(RETRYABLE)`.
- RAG outage → grading runs in degraded mode only if policy permits; otherwise job fails
  cleanly so no ungrounded auto-grade occurs.