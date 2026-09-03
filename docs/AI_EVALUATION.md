# AI Evaluation Lab

A first-class, versioned evaluation layer (ADR-0016) for every AI capability. Golden
datasets, measurable metrics, and regression gates make AI changes reviewable and
auditable — no prompt/model change ships silently.

## 1. Scope & structure

```
evals/
  datasets/<capability>/<set>/    # golden data: inputs + labels + metadata
  config/<capability>.yaml        # gates & thresholds
  results/                        # eval_runs output artifacts (gitignored; DB recorded)
  harness/                        # evaluation code lives in backend/tests or evals utils
```

Capabilities: `rag`, `ocr`, `hw_en`, `hw_ur`, `translation`, `question_generation`,
`grading`.

## 2. Metrics per capability

| Capability | Metrics | Gate (initial) |
|---|---|---|
| RAG | retrieval hit@5, MRR, nDCG@10, relevance (graded), groundedness (citation coverage) | hit@5 ≥ 0.85 per eval set |
| OCR (printed EN/UR) | CER, WER per language | CER ≤ 0.05 (clean set) |
| Handwriting EN | CER, WER | CER ≤ 0.10 |
| Handwriting UR | CER, WER, + "interpretation agreement" (human-verifiable hypotheses) | CER ≤ 0.20 (Phase 2 target) |
| Translation | chrF/COMET-style, terminology retention, human spot-checks | no regression from baseline |
| Question generation | curriculum alignment, answer correctness (manual sample), duplicate rate, difficulty accuracy, Bloom fit, **scope violations (must be 0)**, **weighting adherence** | duplicate rate ≤ 5%; correctness ≥ 90% sampled; scope violations = 0 |
| Grading | teacher-agreement (Cohen's κ), mean absolute error, score agreement, override rate, confidence calibration (ECE) | MAE ≤ 0.5 marks; κ ≥ 0.7 |

## 3. Regression gates

- Every PR touching a prompt, model routing, chunking, OCR pipeline, grading logic, or
  **scope-resolution/coverage logic** runs affected gated datasets.
- Gate result `pass:bool` recorded (`eval_runs`) with thresholds + config hash; failing gate
  blocks merge unless explicitly waived with logged justification.
- Baseline snapshots: `baselines/<capability>.json` — comparisons are against baselines to
  detect drift.

## 4. Datasets lifecycle

- Created via `POST /ai/datasets` (metadata) + artifact upload (`eval_datasets`).
- Versioned; never mutate in place — superseded versions archived.
- Ownership: designated teachers/AI operator per school; platform-level default sets.

## 5. Operation

- Users: `ai:admin` (SETUP/run), `ai:read` (view results).
- Runs: `POST /ai/evals/runs` → queued to `q.system` (or `q.embeddings` for rag rebuild
  checks) → results stored; dashboard in [OBSERVABILITY.md](OBSERVABILITY.md).
- Scheduled regression: nightly on staging; on-demand for hotfix path.

## 6. Confidence calibration

- Grading confidence calibration tracked (ECE) over teacher decisions; mis-calibrated
  configs surfaced → threshold tuning proposals.
- OCR confidence similarly calibrated against corrections.