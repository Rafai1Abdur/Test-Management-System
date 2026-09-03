# RAG — Curriculum-Aware Retrieval

RAG is **not** a linear "PDF → fixed chunks → embed → query" pipeline. It is a scoped,
hybrid, curriculum-aware retrieval system layered over the Derived Semantic Knowledge Index
([QDRANT.md](QDRANT.md)).

## 1. Retrieval pipeline (single query)

```
1. Scope (metadata filter)      Order: school/tenant → academic year → assessment period
                                (coverage-aware) → subject → chapter → section → topic →
                                grade → language → authority → content_type → material
                                allowlist → approved_only
2. Dense retrieval              Encode query with registered embedding model
                                (bge-m3 default) → top-k_dense
3. Sparse/lexical retrieval     BM25/SPLADE-style over sparse vectors → top-k_sparse
4. Hybrid fusion                Reciprocal Rank Fusion (RRF) of both candidate sets
5. Rerank                        Cross-encoder (local) or LLM reranker on top ~40
6. Context assembly             Dedupe → order by provenance & relevance → truncate to
                                token budget → attach source metadata
7. Resolve                      chunk_id → rag_chunks (MongoDB) for authoritative text,
                                page numbers, evidence refs
```

## 2. Scope builder

Every caller supplies a `RetrievalScope`:

| Field | Example | Rule |
|---|---|---|
| `school_id` | `...` | mandatory |
| `academic_year_id` | `2026-27` | mandatory |
| `assessment_period_id?` | `q1-2026` | when present: coverage-mode scope resolution applies (ADR-0017) |
| `coverage_mode` | `CUMULATIVE` | scope-mode of the blueprint; CUMULATIVE expands across ordered periods **within the same academic year** (resolution algorithms: EXAM_ENGINE §2b) |
| `approved_coverage_only` | `true` | hard filter: **only `COMPLETED` coverage is generation-eligible** — `NOT_STARTED` and `IN_PROGRESS` are out of scope; `EXCLUDED` is explicitly forbidden even if selected (EXAM_ENGINE §2b) |
| `material_ids[]` | allowlist | exam generation: **only blueprint-selected** materials |
| `subject_id`, `chapter_ids[]`, `section_ids[]` | | curriculum pins, derived from resolved scope |
| `grade`, `language` | `9`, `UR` | filter |
| `authority` | `[PRIMARY, SECONDARY]` | default; `PRACTICE/REFERENCE` excluded unless asked |
| `content_type` | `[CONCEPT, DEFINITION, EXAMPLE]` | optional |
| `approved_only` | `true` | invariant for generation/grading |

## 2b. Syllabus-aware scope resolution (ADR-0017)

Retrieval scope conceptually incorporates:

```
Academic Year + Assessment Period + Subject + Grade
+ Approved Teaching Coverage (chapter statuses; class/section-specific overrides merged
  per chapter with the grade+subject default — ADR-0017)
+ Selected Chapters/Sections (blueprint scope mode + materialized scope)
+ Approved Learning Materials (allowlist)
```

- **Invariant:** `generation_scope ⊆ approved_teaching_coverage ∩ blueprint_scope`
  (hard filter, fail-closed — no chunk outside the resolved scope is retried even if the
  material is APPROVED).
- **Qdrant stays period-independent** (no `assessment_period_id` payload): scoping is
  applied at **filter time** via resolved chapter/section/material ids. Re-index churn per
  term is avoided; the index remains the Derived Semantic Knowledge Index (ADR-0007).
- Scope resolution is a `rag`-adjacent domain service (`ScopeBuilder` consumes a
  `RetrievalScope` with a resolver); evidence records the resolved scope per call.

## 3. Query variants

- **Query points**: raw question/blueprint text ("What is x…"), question skeleton, or
  undergrading student answer. Different encoders/instructions per use case but the same
  index.
- **Multi-query (option)**: expand one query into rewritten variants (teacher-specified or
  LLM-generated), retrieve per variant, fuse; disabled by default (cost/latency).

## 4. Fusion & rerank details

- **RRF**: `score = Σ 1/(60 + rank_i)` across dense + sparse lists. Configurable constant.
- **Rerankers**: local cross-encoder (`bge-reranker` family) for EN/UR; LLM rerank as
  provider option. Rerank output feeds context assembly.
- Context assembly per **token budget** (model-dependent), ordering: (1) highest relevance,
  (2) source proximity (chapter/section), (3) page order; dedupe near-identical texts via
  MinHash/LSH or embedding cosine.

## 5. Retrieval quality instrumentation

Every retrieval call records (into `model_runs` + observability):

- filter details, candidate counts per leg, fusion scores, final top-k
- `retrieval_latency_ms`, collection/model used, `search_success:bool`
- Offline: logged to enable **RAG evaluation** (hit@k, nDCG, MRR) against golden datasets
  ([AI_EVALUATION.md](AI_EVALUATION.md)).

## 6. Question-generation usage (scoped to materials)

The blueprint defines the *only* legal scope. The generator:

1. resolves blueprint → `RetrievalScope` (allowlist materials/chapters/coverage weights)
2. per blueprint cell (chapter × type × difficulty): retrieve `k` concepts on-topic
3. feeds context + cell spec to generation prompt
4. each generated candidate stores `retrieved_chunk_ids[]` + scores (**evidence chain**).

## 7. Grading usage

Grading fetches relevant approved context for the question (same filters, plus
`question_id`-derived scope) so the grader can ground criterion checks in source material;
context is stored in the evidence snapshot.

## 8. Independence & testability

- `rag` domain exposes a pure `retrieve(scope, query, k)` API with no HTTP coupling →
  unit-testable against a Qdrant test instance seeded from `eval` fixtures.
- Modular components: ScopeBuilder, DenseSearcher, SparseSearcher, Fuser, Reranker,
  ContextAssembler — each swappable/overridable via DI (see [TESTING.md](TESTING.md),
  [AI_EVALUATION.md](AI_EVALUATION.md)).

## 9. Failure & degraded modes

- Qdrant unavailable → `RetrievalUnavailable`; generation fails **before** producing
  ungrounded questions; grading policy decides: fail-close (default) vs continue without
  context (never silently).
- A provider embedding timeout → router retries another provider with same dimension;
  if none, job fails retryable.
- Rebuild in progress → reads continue on old alias; new alias activates after switch.