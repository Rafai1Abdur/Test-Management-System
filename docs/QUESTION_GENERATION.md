# Question Generation

Generated questions and answer keys are **candidates** — never trusted implicitly. Every
candidate is validated, deduplicated, rubric-ed, and teacher-reviewed before it can enter
an exam.

## 1. Pipeline

```
Exam Blueprint
→ resolve scope (scope resolver: approved teaching coverage ∩ blueprint scope [ADR-0017];
  materials[] allowlist, chapters, sections, weighting, language — fail-closed if invalid)
→ scoped RAG retrieval (per blueprint cell: chapter × type × difficulty)
→ candidate generation (LLM, structured output, grounded in retrieved context)
→ structural validation (JSON schema per question type)
→ answer validation (key correctness, option uniqueness, distractors sanity)
→ curriculum alignment check
→ difficulty validation (vs blueprint distribution)
→ learning-objective coverage check (vs blueprint Bloom distribution)
→ duplicate detection (vs bank: exact/lexical + semantic)
→ rubric generation (per subjective type; criteria + descriptors)
→ quality pass (secondary review model or rule checks: ambiguity, leakage, bias)
→ persist candidates w/ evidence chain + status=CANDIDATE
→ teacher review → APPROVED (bank) / edited / REJECTED
```

## 2. Grounding & evidence (AI Evidence Chain)

Each candidate stores:
- `source_refs`: materials + chapters/sections + pages used
- `retrieved_chunk_refs`: chunk IDs + texts + scores (evidence blueprint)
- `prompt_version`, model/provider/version, raw response ref
- validation results per stage
- (published exams render `source_refs` when school policy allows "source visible")

## 2b. Structured assessment context (syllabus-aware generation)

The generator never receives a bare "generate a Grade 8 Physics exam" prompt. The domain
service builds a **structured assessment context** ([ADR-0017](ADR/ADR-0017-assessment-periods-coverage-exam-sets.md)):

- academic year + **assessment period** (name, type) + examination set (if any)
- coverage mode (NEW_ONLY / CUMULATIVE / SELECTED_CHAPTERS / FULL_SYLLABUS / CUSTOM) and the
  **materialized scope**: approved chapters/sections (EXCLUDED chapters listed as forbidden)
- learning objectives, approved source materials, language
- chapter/section **weighting** table + tolerance (default ±10 percentage points,
  school-configurable)

Scope validations added to the pipeline:
- **Scope provenance check**: candidate chapter/section refs must lie within the resolved
  scope — out-of-scope candidates are rejected before review (scope is never silently expanded).
- **Weighting adherence**: marks distribution per chapter must match requested weights within
  tolerance; deviations surface as warnings/errors per school config (metrics in §9).
- **Coverage eligibility (MVP)**: only `COMPLETED` coverage is generation-eligible;
  `NOT_STARTED`/`IN_PROGRESS` chapters are out of scope and `EXCLUDED` chapters are
  forbidden even if explicitly selected — a chapter with any of those statuses produces
  zero generated candidates (EXAM_ENGINE §2b).
- **Choice groups** ("attempt any N of M") are generated/validated as explicit blueprint
  `choice_groups`; assembly/publish enforce `attempt_count ≤ question_count` and marks
  accounting for maximum attainable marks (EXAM_ENGINE §3).

## 3. Structural validation (per type)

| Type | Checks |
|---|---|
| MCQ | exactly one `is_correct`, ≥3 options, non-duplicate options, plausible distractors |
| MSQ | ≥2 correct, at least one incorrect |
| True/False | unambiguous stem |
| Fill-blank | exactly one blank marker, acceptable-answers list (synonyms!) |
| Short/Long/Essay | rubric present; answer-key present; Bloom consistent with stems |
| Numerical | numeric answer + unit + tolerance |
| Matching/Ordering | bijective mapping / deterministic order |

## 4. Duplicate detection (semantic + lexical)

- Exact: canonicalized text SHA-256 (`.sha256_canonical`) — pre-index unique.
- Lexical: token n-gram Jaccard over normalized text; windowed search within subject.
- Semantic: embeddings cosine over candidate vs bank (same embedding model as index);
  threshold → flag as NEAR-DUPLICATE for teacher.
- Duplicates → `DERIVED` relation (origin_question_id) or REJECT.

## 5. Rubric generation

- For subjective types: criteria (max 3–6), each with levels/descriptors and marks;
  total = question marks; language matches exam language.
- Rubric is validated (sum == marks; descriptors non-empty); teacher-editable.

## 6. Quality pass

- Self-check/second model: ambiguity, leaked answers (answer in stem), cultural/offensive
  content, grade-appropriate vocabulary, language purity (no mixed EN/UR unless desired).
- Question-level confidence; policy: below threshold → not suggested to exam builder.

## 7. Teacher review workflow

`GET /api/v1/questions?status=CANDIDATE` → approve/edit/reject. Edits create new revision
(versioned); review history audited. Only APPROVED questions are bank-searchable and
exam-usable.

## 8. Scales & throughput

- Parallel generation per blueprint cell (worker pool `worker-generation`);
  idempotent per cell job; retries on provider failure; per-cell progress in `jobs`.

## 9. Eval & metrics

- Golden QG dataset: curriculum-aligned pairs; offline eval before prompt/model promotion
  ([AI_EVALUATION.md](AI_EVALUATION.md)) — curriculum alignment, correctness, duplicate
  rate, difficulty accuracy, **scope-violation rate (must be 0)**, **weighting adherence**,
  review-edit rate as live signal.