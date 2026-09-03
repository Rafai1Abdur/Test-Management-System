# ADR-0016 — AI Evaluation Lab (Golden Datasets + Regression Gates)

- **Date:** 2026-09-02
- **Status:** Approved

## Context

AI capabilities (RAG, OCR, EN/UR handwriting, translation, question generation, grading)
change constantly via prompts, models, and pipelines. Without measured quality gates,
production quality silently drifts and regressions ship unnoticed. The master project spec
requires evaluation datasets & metrics for each capability.

## Decision

1. **AI Evaluation Lab is a first-class component** (AI_EVALUATION.md): golden datasets per
   capability (versioned, `eval_datasets`), measurable metrics, and **regression gates**
   recorded in `eval_runs`.
2. Metrics defined per capability (RAG hit@k/MRR/nDCG; OCR/handwriting CER/WER;
   translation chrF/COMET + human spot-checks; QG curriculum alignment/correctness/
   duplicate rate; grading MAE/κ/override/calibration).
3. PRs touching prompts, routing, chunking, OCR pipelines, or grading logic **must pass**
   affected gates (or be explicitly waived with a logged risk decision).
4. Baselines stored for drift comparison; schedules: nightly regression on staging,
   on-demand for hotfix.
5. Datasets are in `evals/datasets/` (planned repo layout); results in `eval_runs` + CI.

## Consequences

- **Positive:** measurable quality, safe model/prompt rollouts, calibration feedback loop,
  defensible thresholds.
- **Negative:** dataset curation effort (owned per phase — RAG Phase 3, OCR Phase 6, HW
  Phase 7, QG Phase 4, grading Phase 8); CI time for gated runs.

## Invalidation triggers

Engineering capacity constraints → gates can be made advisory per capability with a
recorded decision (never silently dropped).