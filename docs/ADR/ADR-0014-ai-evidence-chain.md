# ADR-0014 — Mandatory AI Evidence Chain

- **Date:** 2026-09-02
- **Status:** Approved

## Context

Teachers must be able to understand *why* the system generated a question or awarded a
score, and auditors must have complete provenance for every AI decision. Without
structured evidence, enrichment (retrieval, reasoning) is unverifiable and questions of
quality cannot be debugged.

## Decision

1. **Every AI artifact carries an evidence chain** — an immutable record of its inputs and
   reasoning:
   - Question/key/rubric: blueprint; selected materials; retrieved chunk refs (+texts +
     scores); chapter/section/page; prompt version; model/provider/version; raw response
     ref; validation results.
   - Grading: question; official answer; rubric; retrieved curriculum context; OCR result +
     crop refs; translation refs; prompt version; model run ids; per-criterion reasoning;
     candidate score; confidence.
   - OCR/handwriting/translation: crop refs; engines; raw outputs; consensus votes;
     confidence; normalization details.
2. Evidence stored two ways: inline `evidence` object (hot path) and immutable
   `evidence_snapshots` collection (complete package, platform-lifetime retention).
3. Evidence is **append-only/immutable**: teacher edits create new records referencing
   originals; never mutate.
4. Evidence is surfaced in the verification UI (TEACHER_VERIFICATION.md) and used by eval
   workflows.

## Consequences

- **Positive:** explainability, auditability, debuggability of AI problems, retraining
  datasets; trust.
- **Negative:** storage growth (controlled by snapshot retention + refs without duplicating
  large blobs); schema discipline required.

## Invalidation triggers

Product simplification dropping explainability (unlikely; core team principle).