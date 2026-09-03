# ADR-0012 — AI Output Is Never the Final Authority

- **Date:** 2026-09-02
- **Status:** Approved (core architectural principle)

## Context

AI systems produce OCR text, handwriting interpretations, translations, questions, answer
keys, and grades. None of these should silently be treated as truth: errors cascade into
wrong student outcomes. Product rule requires confidence scoring and teacher verification.

## Decision

**Core principle: AI output is NEVER automatically the final authority.** Uniformly
applies to OCR, handwriting interpretation, translation, question generation, answer-key
generation, and grading (AI_ARCHITECTURE.md §2).

1. Every AI result uses the standard envelope: `value` (candidate), `confidence`,
   `status` ∈ {CANDIDATE, NEEDS_REVIEW, APPROVED, REJECTED}, provider/model/version,
   prompt_version, evidence ref.
2. **Confidence gates** are configurable per task × language (defaults in
   AI_ARCHITECTURE.md §2.2; Urdu handwriting strictest); below gate → mandatory teacher
   review; even above gate, teacher approval is required by policy (bulk-accept is an
   explicit teacher action — never automatic finalization).
3. Objective-question grading (deterministic rule vs teacher-approved key) is **not AI**
   output and may auto-finalize (audited).
4. Candidate and final values live in separate fields; final set only by teacher-approved
   paths (domain-enforced).
5. **Generated exam scope is an AI-adjacent decision with the same discipline:** an exam's
   syllabus/scope must equal the approved assessment context (approved teaching/curriculum
   coverage ∩ blueprint scope, ADR-0017). When AI proposes questions, its evidence chain
   records the scope provenance, and out-of-scope candidates are rejected before review —
   never silently included.

## Consequences

- **Positive:** correctness and trust; auditability; defensible results; safe defaults for
  Urdu/uncertain handwriting; measurable override rates.
- **Negative:** teacher workload (mitigated by bulk-accept + quality gates + eval-driven
  threshold tuning).

## Invalidation triggers

Only by deliberate product-level change via a new ADR (not anticipated).