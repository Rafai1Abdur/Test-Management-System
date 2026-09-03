# Cost Management (AI Spend)

## 1. Ledger

Every AI invocation is recorded (`model_runs`) and contributes an `ai_usage` ledger row:

| Field | Example |
|---|---|
| provider / model / model_version | `openai / gpt-4o-mini / 2026-06-01` |
| operation | `question_generate` |
| school_id / user_id | tenant attribution |
| input_tokens / output_tokens | 1200 / 310 |
| images: {count, est_pixels} | OCR VLM prompt |
| latency_ms | 2800 |
| estimated_cost (currency) | computed from registry cost_profile |
| timestamp / batch job | — |

Cost estimation uses registry `cost_profile` (input/output per 1M, image factor); actuals
from provider billing APIs can be imported to reconcile.

## 2. Budgets & alerts

- Per-school monthly budget (platform config); per-task multipliers.
- Alerts at 70/90/100% of budget (Grafana + notification); hard block (optional) at 100%
  for cost-heavy tasks (question generation, VLM OCR) while grading stays allowed.
- Routing affects spend: MODEL_ROUTING tiers (BALANCED vs BEST); local-first schools can
  run near-zero marginal cost (Ollama/bge-m3/Tesseract).

## 3. Reports

- `/ai/usage` — monthly by (school, operation, model), token trends, cost trends,
  top consumers, per-exam cost estimates (planned vs actual).
- Cost attached to evidence (`model_runs` refs) for full traceability.

## 4. Optimization levers

| Lever | Effect |
|---|---|
| model tier per task | cheaper where quality holds |
| caching embeddings (content-hash → vector) | avoid re-embed |
| batch/parallel within token caps | better latency/price ratio |
| OCR first-pass local (Tesseract/Paddle) | avoid VLM unless needed |
| question-draft validation via cheaper model | save expensive calls |
| eval-driven model swap | quality/cost frontier review |

## 5. Governance

- Cost data is audit-retained (monthly aggregates indefinite; raw rows 1 year archived).
- Admin-only visibility; per-school principals see school totals.
- Open question: no real billing import yet — planned Phase 11 hook.