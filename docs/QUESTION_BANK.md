# Question Bank

A reusable, versioned, curriculum-tagged question bank with usage and performance stats.

## 1. Entity model

`questions` (bank entries) + `question_versions` (immutable history) +
`question_usage_stats` (rollups). Full schema in [MONGODB_SCHEMA.md](MONGODB_SCHEMA.md).

Fields carried per question: type, subject/grade/chapter/section/topic, difficulty,
Bloom level, marks, text, options, correct answer, explanation, rubric, source refs,
creation method (`AI|MANUAL|IMPORT`), AI model/prompt version, version, approval state,
usage stats, performance stats.

## 2. Ordering principle

The bank is a **shared library per school**, not a per-exam scratch space. Questions can be
imported (AI), authored (manual), or generated then approved. Approvals are per school.

## 3. Versioning

- Every edit creates a new revision (`rev++`, snapshot in `question_versions`); exam
  versions reference approved question snapshots, so published exams are unaffected by later
  question edits.
- Deprecation: `DEPRECATED` (hidden from pickers) keeps history.

## 4. Duplicate detection (semantic + lexical)

- Lexical: canonicalized-text hash (exact) + n-gram Jaccard (near-exact).
- Semantic: embedding cosine (same index/model as RAG) — near-duplicate flags for teacher.
- On duplicate: choose REJECT (keep single source) referencing `origin_question_id`, or
  use-case variants (`USAGE_VARIANT`).

## 5. Search & filtering (teacher picker)

Filter by: subject/grade/chapter/section/topic, question type, difficulty, Bloom,
language, marks, approval state, creation method, usage count, performance (p-value),
source material. Paginated; sortable; returns summaries w/o correct answers unless
key-permitted.

## 6. Usage & performance stats

- `times_used`, `times_correct`, `p-value`, `discrimination_index` (when data permits),
  `avg_time` (optional), per-chapter/class breakdowns.
- Feeding analytics ([OBSERVABILITY.md](OBSERVABILITY.md), [ROADMAP.md](ROADMAP.md)).

## 7. Integrity & privacy

- Correct answers and answer keys are RBAC-gated; question lists sent to exam printing
  exclude answers ([EXAM_INTEGRITY.md](EXAM_INTEGRITY.md)).
- Question content copyright: school-managed; usage logged.