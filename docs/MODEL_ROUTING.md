# Model Routing

The router maps a **task** (+ context: language, budget, latency budget, quality tier) to a
concrete **model** in `ai_model_registry` / `embedding_model_registry`. It is the only place
that resolves "which model do I call?" — domain code stays model-agnostic.

## 1. Task → capability mapping

| Task | Capability | Considerations |
|---|---|---|
| Question generation | structured + text | factual grounding (EN/UR), JSON schema, price/latency |
| Answer-key & rubric generation | structured | strict correct-answer validation |
| Document analysis (subject/grade/chapter detection) | vision + text | pick vision-capable if scanned; else text |
| Grading subjective | reasoning + text | strongest model per budget tier; evidence + explanation required |
| OCR printed | ocr | Tesseract/PaddleOCR (free/local); VLM fallback |
| Handwriting (EN/UR) | handwriting | multi-model consensus (see HANDWRITING.md) |
| Translation | translation | language pair aware |
| Embeddings | embeddings | model dimension/version must match Qdrant collection |
| Rerank | rerank | cross-encoder preferred; LLM fallback |

## 2. Selection inputs & policy

Scored selection over eligible ACTIVE models (`capability ∈ task`, `language ∈ model`,
`status=ACTIVE`):

1. **Capability + language/vision/structured support** (hard filter)
2. **Quality tier** from platform/school config (default `BALANCED`) — weights:
3. Cost (from `cost_profile`) — soft weight
4. Latency budget (task-level) — soft weight
5. Availability/health (provider heartbeat, circuit status) — hard filter when down
6. Data-policy match (local vs cloud — e.g., never send student answers to a cloud provider
   for a school that opted out) — **hard filter (privacy-aware routing)**

Routing decisions logged in `model_runs` (provider/model chosen, alternates considered,
reason code) for audit + cost.

## 3. Fallbacks & retries

- Define an `eligibility chain` per task (config): e.g., grading:
  `[gemini-flash(local-opt-in) → gpt-4o-mini → ollama-local:qwen]`.
- Router retries on transient failure across the chain (max N hops), logging each hop.
- No eligible model → job → `FAILED(RETRYABLE)` with reason; never silent degradation into
  an unauthorized provider (potential data-policy violation).

## 4. Embedding routing (index-consistency constraint)

- **Embedding model is pinned per Qdrant collection** (invariant). The router cannot switch
  embedding models mid-flight. Resolution:
  1. Resolve the *collection's* registered `embedding_model_id` (from
     `embedding_model_registry` + Qdrant alias).
  2. Route the embed call to a provider serving *that exact model+version* (local/vLLM
     preferred).
  3. Changing embedding model = a **versioned re-index event** (QDRANT.md §8), never a
     silent router switch.
- bge-m3 is the MVP default; registry-driven; documented hard-swap path exists.

## 5. Configuration shape

```yaml
routing:
  default_tier: BALANCED
  task_profiles:
    grading_subjective:
      capability: reasoning
      tier: BEST        # highest-quality, cost-aware
      eligibility: [gemini-flash, gpt-4o-mini, ollama/qwen2.5]
      max_hops: 2
      timeout_s: 60
    question_generation:
      capability: structured
      tier: BALANCED
      eligibility: [gemini-flash, glm-4, gpt-4o-mini]
      ...
    embeddings:
      pin_to_collection: chunks_live
      ...
  data_policy:
    cloud_allowed: false    # school-level opt-in for cloud providers
    student_answers_cloud: false
```

## 6. Observability & governance

- Router emits per-request decision telemetry (model, reason, latency, cost) —
  [OBSERVABILITY.md](OBSERVABILITY.md).
- Admin can pin/pause models (e.g., after a provider outage or a pricing change); pinned
  changes are audited.
- Eval-driven guardrail: a model can be configured `eval_gated: true` so traffic to it is
  shadow-logged for the first N runs (safe rollout) before becoming eligible.