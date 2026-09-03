# AI Architecture — Capabilities, Gateway, Evidence, and the "Never Final" Invariant

## 1. Capability model (capabilities are orthogonal to providers)

The system defines **AI capabilities**, not provider-specific APIs:

| Capability | Used by | Output |
|---|---|---|
| `text` | question/key/rubric generation, analysis | text |
| `structured` | JSON-schema-constrained generation (blueprint cells, rubric criteria) | typed JSON |
| `vision` | document analysis, diagram Qs, photo pre-processing | labels/descriptions |
| `ocr` | printed text extraction (EN/UR) | text + boxes + confidence |
| `handwriting` | handwritten answer interpretation (EN/UR Phase 2) | text + confidence |
| `translation` | answer translation between supported languages | translated text |
| `embeddings` | dense vectors for Qdrant | vectors |
| `rerank` | hybrid retrieval rerank | scores |
| `reasoning` | grading justifications, chain-of-thought candidates | text |

A provider may support one or many capabilities; the **model registry** records them and
the **router** picks a model for a (task, language, budget) per request
([MODEL_ROUTING.md](MODEL_ROUTING.md), [AI_PROVIDER_GATEWAY.md](AI_PROVIDER_GATEWAY.md)).

## 2. Core invariant: AI output is NEVER automatically the final authority

Applies uniformly to **OCR, handwriting interpretation, translation, question generation,
answer-key generation, and grading** (ADR-0012).

### 2.1 Standard AI-result envelope

```json
{
  "value": "...",
  "confidence": 0.0 .. 1.0,
  "status": "CANDIDATE",           // CANDIDATE | NEEDS_REVIEW | APPROVED | REJECTED
  "verification_policy": "REVIEW_REQUIRED" | "BULK_ACCEPTABLE",
  "provider": "...",
  "model_id": "...",
  "model_version": "...",
  "prompt_version": "...",
  "evidence_ref": "..."
}
```

`value` is always a **candidate** until a human (or a certified deterministic rule —
objective grading) transitions it to APPROVED.

### 2.2 Confidence gates (configurable per task × language)

| Task | Default gate (below → mandatory review) | Notes |
|---|---|---|
| Printed OCR (EN/UR) | 0.80 | Region-level; low → teacher repair in review UI |
| Handwriting EN | 0.70 | Phase 2 default; stricter than printed |
| Handwriting UR | 0.60 | **Strictest default**; near-consensus still reviewed |
| Translation | 0.70 | Preserve original regardless |
| Question/key/rubric generation | human-review step always | no auto-add to exams |
| Subjective grading | 0.85 | auto-acceptable only via bulk-approve by teacher |

All thresholds live in platform config (per-school overridable) and are recorded on each
grading/OCR result row. Gates are non-bypassable in code paths (enforced by domain layer).

### 2.3 What this means in flows

- **Objective questions** (MCQ/TF): grading is deterministic rule application against the
  teacher-approved answer key → NOT an AI output → may auto-finalize (still audited).
- **Everything AI-produced**: enters a candidate state; nothing flows to results/print
  without teacher action according to policy.

## 3. AI Evidence Chain (ADR-0014)

Every AI artifact is shipped with an immutable evidence package answering
**"why did the system do X?"**. Two representations:

- **Inline `evidence` object** on questions/grading_results (hot path, includes refs).
- **`evidence_snapshots`** collection (complete immutable package, platform-lifetime).

Evidence package contents for each producer:

| Producer | Evidence includes |
|---|---|
| Question generation | blueprint id; selected materials; retrieved chunk refs (+texts, scores); chapter/section/page; prompt version; model+provider; raw response ref; validation results |
| Answer key / rubric | same as question + rubric criteria source rationale |
| Grading | question; official answer; rubric; retrieved curriculum context; OCR result + crop ref; translation; prompt version; model run ids; per-criterion reasoning; candidate score; confidence |
| OCR/handwriting | crop refs; engines run; raw outputs; consensus votes; confidence per region |
| Translation | source (original) ref; target text; confidence; model info |

Evidence is **append-only/immutable**; teacher edits create new records that reference (not
modify) the originals. Teacher-facing UI surfaces evidence in the verification workspace
([TEACHER_VERIFICATION.md](TEACHER_VERIFICATION.md)).

## 4. Registry & versioning

- `ai_model_registry` / `embedding_model_registry` collections (see MONGODB_SCHEMA.md).
- **Prompt versioning**: every prompt lives in versioned files
  (`backend/app/domains/ai/prompts/<capability>/<name>@<version>.j2`); `prompt_version` is
  recorded on every run; prompts are regression-evaluated before rollout
  ([AI_EVALUATION.md](AI_EVALUATION.md)).
- Model configuration changes (temperature, JSON mode, timeout) are versioned with the
  registry row.

## 5. Run recording, cost & audit

Every invocation of any AI capability writes:

1. `model_runs` record (provider, model, tokens, latency, cost, operation, refs)
2. cost ledger entry (`ai_usage`) for [COST_MANAGEMENT.md](COST_MANAGEMENT.md)
3. evidence snapshot for high-stakes artifacts
4. audit event (who/what/context)

## 6. Provider independence

- Domain code depends only on capability interfaces from `domains/ai/gateway`.
- Providers are adapters (OpenAI, Gemini, Anthropic, GLM, Ollama, vLLM, local Tesseract/
  PaddleOCR, VLM adapters). No provider-specific branching in business logic. See
  [AI_PROVIDER_GATEWAY.md](AI_PROVIDER_GATEWAY.md).

## 7. Quality & evaluation

- All capabilities are evaluated in the **AI Evaluation Lab**
  ([AI_EVALUATION.md](AI_EVALUATION.md)) before rollout and on every change (regression
  gates). Prompts/models cannot be promoted without an eval pass unless explicitly bypassed
  with a logged risk decision.
- Production traffic produces continuous metrics (confidence calibration, teacher override
  rates, retrieval hit-rate proxies) in [OBSERVABILITY.md](OBSERVABILITY.md).