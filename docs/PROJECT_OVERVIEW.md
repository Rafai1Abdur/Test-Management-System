# Project Overview — AI-Powered School Test & Examination Management System

## Vision

A scalable educational assessment platform that guides a school from **raw teaching
materials** (textbooks, notes, worksheets) all the way to **graded, verified student
results** — with curriculum-aware AI assistance everywhere, but with **humans always in
control of final outcomes**.

The system must work for schools whose working languages include **English and Urdu**,
handle **printed and handwritten** student work, produce **printable and machine-readable**
exams, and remain **operable on a single on-prem/VPS server** for a first school while being
**architecturally ready to scale** horizontally.

## Non-negotiable goals

1. Every AI result (OCR, handwriting interpretation, translation, question generation,
   answer-key generation, grading) is a **candidate**, never the final authority.
2. MongoDB is the system of record; Qdrant is the Derived Semantic Knowledge Index;
   object storage holds the original artifacts.
3. No lock-in: AI providers, embedding models, and OCR engines are pluggable.
4. Full auditability: every AI decision can be explained (AI Evidence Chain).
5. Frontend and backend are physically separated.
6. MVP is a modular monolith + Celery workers on Docker Compose — no microservices.

## Actors / roles

| Role | Who | Primary interests |
|---|---|---|
| Super Admin | Platform operator | Manage schools, tenancy, budgets, global config |
| Admin | School IT/admin | Manage users, academic years, classes, subjects |
| Principal | School leadership | Oversight, approvals, analytics, material authority |
| Exam Coordinator | Staff | Build blueprints, publish exams, manage integrity |
| Teacher | Staff | Upload materials, approve content, generate exams, grade, verify |
| Student | Learner | Take exams (paper), receive results |

Full RBAC matrix: [AUTH_RBAC.md](AUTH_RBAC.md).

## Core workflows

1. **Curriculum setup** — school → academic year → classes/sections → subjects → chapters →
   topics. ([SYSTEM_DESIGN.md](SYSTEM_DESIGN.md))
2. **Assessment calendar & coverage** — create assessment periods within the academic year
   (First/Second/Third Term, Quarter 1–3, Mid-Term, Final Term, custom); record which
   chapters/sections were taught per period (teaching/curriculum coverage; chapter-level
   mandatory, topic-level optional; DRAFT → REVIEWED → LOCKED). See
   [EXAM_ENGINE.md](EXAM_ENGINE.md) §2b and [MONGODB_SCHEMA.md](MONGODB_SCHEMA.md).
3. **Material ingestion** — teacher uploads a book/PDF/DOCX/PPT/image; pipeline validates,
   extracts, analyzes, chunks, embeds, indexes, and submits for approval.
4. **Examination sets** — group the exams of a period (e.g., Grade 8 Mid-Term Set:
   Mathematics, Physics, English, Urdu, Chemistry) with grade/class scope, subjects,
   schedule (advisory in MVP), and publication state.
5. **Curriculum-aware RAG** — **syllabus-aware** scoped retrieval over approved materials:
   scope = Academic Year + Assessment Period + Subject + Grade + approved teaching coverage
   + selected chapters/sections + approved learning materials. Retrieval is filter-time
   scoped; the period/coverage resolution never changes the underlying Qdrant index.
6. **Question generation** — from an exam blueprint (which carries the resolved,
   coverage-validated syllabus), generate candidates grounded in RAG context, validate,
   weight-check, dedupe, generate rubrics, submit for teacher review. Out-of-scope
   candidates are rejected before review.
7. **Exam building & printing** — teacher assembles exam from blueprint/question bank,
   publishes a locked version (syllabus frozen), prints PDF/DOCX with QR/barcoded paper
   instances.
8. **Student paper processing** — upload scans/photos → page detection → correction →
   answer-region detection → OCR/handwriting → translation → answer reconstruction.
9. **Grading & verification** — objective questions graded deterministically; subjective
   graded by AI with rubric + evidence + confidence; uncertain items queued for teacher.
10. **Results & analytics** — teacher-approved scores propagate to student/class/subject/
    chapter analytics, **reportable per assessment period / examination set**

```
Academic Year
→ Assessment Period (e.g., Quarter 1, Mid-Term, Final Term)
→ Teaching/Curriculum Coverage (chapters/sections taught per subject)
→ Examination Set (grade's period exams)
→ Individual Exam → Exam Blueprint → Exam Version → Exam Paper Instance
```

## Scope boundary of the first release (MVP)

### In MVP (Phase 1–6 slice, see ROADMAP)
- One school; users (admin/principal/teacher/exam-coordinator/student); academic years;
  classes/sections; subjects; chapters/topics.
- **Assessment periods** within the academic year (quarters/terms/mid-term/final/custom);
  **teaching/curriculum coverage** recording (chapter-level mandatory, topic-level optional,
  DRAFT → REVIEWED → LOCKED); **examination sets** grouping each period's exams (advisory
  scheduling).
- Material upload for **PDF, DOCX, TXT, Markdown, HTML, JPEG/PNG**; processing pipeline;
  teacher approval; source authority (PRIMARY/SECONDARY/REFERENCE/PRACTICE).
- Curriculum-aware RAG over approved materials (dense + sparse hybrid, metadata filters).
- Question bank + generation for **MCQ, True/False, Fill-in-the-blank, Short Answer,
  Long Answer**; rubric generation; duplicate detection; teacher review.
- Exam blueprints; exam assembly; **PDF** output with QR/barcodes and randomized paper
  instances; publication locking.
- Student paper upload (multi-page), printed-answer-region detection, **printed Text OCR
  (English + Urdu)** with Tesseract/PaddleOCR adapters, answer reconstruction.
- Objective auto-grading; AI rubric-based grading for subjective answers (short/long) with
  evidence chain + confidence; teacher verification queue; final scores.
- Basic results and analytics (student/class/question averages, pass rate).
- Audit trail, job queue, object storage, backups, cost ledger.

### Phase 2 (documented, not in MVP)
- DOCX exam generation; bilingual (EN/UR) exam layouts; translation pipeline for answers;
  matching/ordering/numerical question types; advanced analytics (chapter mastery,
  discrimination); Urdu+English **handwriting** with consensus + verification gates.

### Future (architecturally defined, designed-in from day one)
- Additional languages; diagram/equation/table question types and grading; multi-school
  platform features; horizontal scale-out; Kubernetes path; advanced reporting.

> **Urdu handwriting note:** Urdu handwriting recognition is *architecturally supported from
> day one* (adapters, consensus, confidence, evidence, verification) and appears in Phase 2.
> It is never claimed to be perfect — it always runs behind confidence gates and mandatory
> human verification below configurable thresholds. See [HANDWRITING.md](HANDWRITING.md).

## Open questions

| # | Question | Affects | Resolved by |
|---|---|---|---|
| 1 | Which LLM providers/keys are available (GLM, OpenAI, Gemini, Anthropic, Ollama/vLLM local)? | MODEL_ROUTING defaults | Phase 3+ |
| 2 | Urdu textbook publisher/board (e.g., PCTB)? | CHUNKING metadata taxonomy | Phase 2 |
| 3 | Copyright/licensing of uploaded textbooks? | SECURITY | Phase 1 |
| 4 | Scale targets (students/school, exams/term) | Indexes, worker sizing | Phase 1 |
| 5 | Handwriting volume per exam (all subjective or spot checks)? | OCR worker pool, cost | Phase 6–7 |
| 6 | Per-student unique paper printing required for all exams or only finals? | EXAM_INTEGRITY scope | Phase 5 |

## Releases

See [ROADMAP.md](ROADMAP.md) for the phased plan (Phase 0 = this documentation; Phase 1 =
foundation).