# Chunking — Hierarchical & Semantic

Chunking produces the canonical `rag_chunks` records (MongoDB) that back the Qdrant index.
Chunking is **hierarchical and semantic**, driven by detected document structure — not by
fixed character windows.

## 1. Chunk hierarchy

| Level | Granularity | Example |
|---|---|---|
| `DOCUMENT` | Whole material | `Book: Grade 9 Biology (PCTB)` |
| `BOOK/PART` | Volume/part | `Part II` |
| `CHAPTER` | Chapter | `Ch.5 — Cell Cycle` |
| `SECTION` | Section within chapter | `5.2 Mitosis` |
| `TOPIC` | Topic/subsection | `Prophase stages` |
| `CONCEPT/DEFINITION/EXAMPLE/EXERCISE/FORMULA/NOTE` | Atomic learning unit | a definition, a solved example, an exercise block |

Chunks are created at the **atomic level**, then **parent-joined upstream** for context.
Each chunk stores `hierarchy_path` (breadcrumbs) so retrieval can expand/contract.

## 2. Extraction-driven segmentation

1. **Structure detection** (per format adapter): headings, TOC, numbering, styles
   (DOCX outline levels; PDF text-with-geometry; EPUB chapter markers; HTML h1–h6;
   PPTX slide titles).
2. **Unit boundaries** inferred from detected structure + semantic shifts (LLM-assisted
   boundary proposal flagged `CANDIDATE`, teacher can correct).
3. **Per-unit submit**: `rag_chunks.create` with `content_type` + parent refs +
   `learning_objective` (from curriculum when mappable) + `difficulty` heuristic
   (from curriculum/verb ontology).

## 3. Chunk quantity bounds (guideline)

- Atomic chunk ≤ ~300 words / ≤ 800 tokens (dense embedding window-comfortable).
- Larger units: keep atomic, link via `parent_chunk_id` (avoid giant blobs).
- Exercises marked `EXERCISE` and not split further unless subproblems matter.

## 4. Metadata attached to every chunk (payload)

`chunk_id`, `material_id`, `subject_id`, `chapter_id`, `section_id`, `topic_id`,
`grade`, `language`, `content_type`, `source_type`, `authority` (inherited from material),
`page_number`, `learning_objective`, `difficulty`, `hierarchy_path[]`, `embedding_model_*`.

Chunks are **period-independent**: assessment periods and teaching coverage are never chunk
payload fields — syllabus scoping is applied at retrieval time via Qdrant filters
([RAG.md](RAG.md) §2b), avoiding re-index churn per term ([ADR-0017](ADR/ADR-0017-assessment-periods-coverage-exam-sets.md)).

## 5. Language handling

- Detect per-unit (or per-document) language; store `text_original_lang` (e.g. `UR`) and
  `text`; embed in **original language** (bge-m3 supports EN/UR). Optionally index a
  normalized/transliterated mirror for cross-language search (Phase 2+).
- Never merge units of different languages into one chunk.

## 6. Semantics-aware refinement

- Definition sentences → `DEFINITION` chunks.
- Worked problems + rules → `EXAMPLE` / `FORMULA`.
- Question sets → `EXERCISE`.
- Narrative paragraphs → `CONCEPT` (kept small when cohesive).

## 7. Versioning & re-chunking

- Chunk set belongs to a **material version**. Editing the material (new edition) → new
  chunk batch → old chunks `is_active=false`, Qdrant points tombstoned; evidence that
  references old `chunk_id`s remains valid (immutability).
- `rag_chunks` + `material_processing_jobs` record who/when/what re-chunked.

## 8. Quality gates

- Coverage: every page/heading accounted for; zero-length / garbage chunks rejected.
- Similarity audit: intra-material near-duplicate chunk rate flag.
- Human sample review: teacher can approve/reject chunk boundaries on NEEDS_REVIEW
  materials (configurable sampling).

## 9. Evaluation hooks

Chunk quality is measurable: golden dataset `rag` contains judged "needle" units;
retrieval recall per topic is regression-gated ([AI_EVALUATION.md](AI_EVALUATION.md)).