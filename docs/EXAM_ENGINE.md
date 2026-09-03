# Exam Engine — Blueprints, Domain Model, Exam Types, Versions

## 1. Exam types & grading-rule variance

| Type | Marks | Duration | Auto-grade | Notes |
|---|---|---|---|---|
| Quiz | small | short | objective yes | quick feedback |
| Class Test | medium | single period | objective + subjective slack | typical school test |
| Monthly Test | medium | 1–2 periods | mixed | |
| Midterm / Final | high | full session | mixed w/ verification | blueprint-driven, strict integrity |
| Mock | like final | like final | like final | practice |
| Assignment / Homework | variable | multi-day | optional | no paper pipeline needed |
| Practical | variable | lab | rubric-based | rubric-first grading |
| Viva / Oral | variable | session | manual w/ rubric aid | voice not processed in MVP |

Grading rules are **per-type policies** (which question types allowed, auto-grade on/off,
verification depth) — not assumptions baked into code.

## 2. Exam blueprint (struct spec)

Fields (schema in [MONGODB_SCHEMA.md](MONGODB_SCHEMA.md) `exam_blueprints`):
school, **academic year, assessment period, examination set**, grade, subject,
**selected materials allowlist**, **coverage mode** (NEW_ONLY | CUMULATIVE |
SELECTED_CHAPTERS | FULL_SYLLABUS | CUSTOM), **teaching-coverage constraint (approved
coverage only)**, chapters, sections, **chapter/section weighting + tolerance (±10
percentage points default, absolute)**, **learning objectives**, total marks, duration,
per-type question specs (count × marks × difficulty dist × Bloom dist), language,
instructions.

The generator treats the blueprint as a **constraint program**: solve the
coverage/weighting/mark matrix, per cell retrieve context (syllabus-aware RAG), generate
candidates, validate, assemble, then present for teacher selection/review. Generation is
**fail-closed** on scope: `generation_scope ⊆ approved_teaching_coverage ∩ blueprint_scope`.

## 2b. Assessment periods, teaching coverage, examination sets & scope (ADR-0017)

**Distinct concepts (never merged):**

| Concept | Meaning | Examples |
|---|---|---|
| Assessment Period | curriculum/time window | Quarter 1, Mid-Term, Final Term, First/Second/Third Term |
| Exam Type | assessment classification | Quiz, Monthly Test, Midterm, Final, Class Test |
| Examination Set | grouping of a period's exams | Grade 8 Mid-Term Set |
| Exam | individual subject assessment | Grade 8 Mathematics (Mid-Term) |

**Hierarchy:** `Academic Year → Assessment Period → Teaching/Coverage → Examination Set →
Exam → Blueprint → Version → Paper Instance`.

**Teaching coverage:** per (year, period, subject, *optional* class/section), chapters
recorded `NOT_STARTED|IN_PROGRESS|COMPLETED|EXCLUDED` (chapter-level mandatory; topic-level
optional), state `DRAFT → REVIEWED → LOCKED`. Resolution inheritance: class/section-specific
coverage wins; otherwise grade+subject default applies.

**Examination set:** groups a period's exams (grade, class/section scope, subjects,
advisory schedule, publication state). Publishing a set does not publish member exams.

**Scope resolution (materialize + revalidate):**
1. Blueprint creation → `resolve-scope`: resolve requested scope from approved/**locked**
   coverage; materialize resolved chapters/sections into the blueprint (teacher reviews /
   edits while DRAFT).
2. Generation → revalidate materialized scope against applicable coverage; **fail closed**
   if invalid (e.g., coverage retracted); never silently expand.

**Syllabus lock:** on exam READY, the scope/coverage is frozen; on PUBLISHED it is fully
locked. Post-publication coverage changes require a new exam version — never silent mutation.

## 3. Canonical exam domain model (single source of truth)

```
Exam
 ├─ header: exam_id, name, version, exam_type, subject, grade, class_ids,
 │           assessment_period_id, examination_set_id, blueprint_id, syllabus_snapshot,
 │           duration, total_marks, language(s), instructions, watermark/branding config
 ├─ sections: [ {title, instructions, questions: [QuestionRef...] } ]
 ├─ question instances: [ {question_id, ref_version, marks, order (per paper), options_order} ]
 ├─ key: ExamKey (separate, RBAC) — options_correct, official answers, rubric ids
 ├─ papers config: randomization on/off, seeds
 └─ integrity: qr_payload template, checksums
```

- **PDF and DOCX are both derived from this model** via the template engine
  ([PDF_DOCX.md](PDF_DOCX.md), ADR-0011) — never independently authored.
- The model is versioned; published exam version = immutable snapshot + checksum.

## 4. Exam lifecycle & versions

```
DRAFT → READY (syllabus frozen) → PUBLISHED (locked) → ARCHIVED
         │          │
         │          └─ corrections → v2,v3... (immutable each)
         └─ preview/generate limited (answer keys withheld)
```
- Draft editing audit-trailed; publish validates: all Qs APPROVED, key APPROVED, marks sum
  == total, **scope/weighting within tolerance**, blueprint coverage met (warnings for soft
  deviations), materials APPROVED, **syllabus lock applied**.
- Each published version stores `checksum_sha256` + **scope_snapshot (coverage hash)** +
  change notes + who/when.

## 5. Answer key lifecycle

- Key created/populated from approved bank questions (or hand-authored).
- Key approval (`key:approve`) required before publish.
- Key correction post-publish → requires new version (integrity).

## 6. Assembly rules

- Assembly is a worker step (`worker-export` not needed; `worker-generation` assembles)
  but rendering is export workers.
- Blueprint soft constraints (coverage < target) produce warnings for the teacher, not
  silent passes.

## 7. API & frontend

See [API.md](API.md) `/exam-blueprints`, `/exams`, `/papers` and
[PDF_DOCX.md](PDF_DOCX.md) for rendering. UI is teacher-workflow-first ("builder"
inspired by blueprint cells, not DB tables).