# Grading

## 1. Two regimes

| Regime | Mechanism | Finality |
|---|---|---|
| **Objective** (MCQ, TF, Matching, Ordering, Fill-blank w/ exact key) | Deterministic compare against teacher-**approved** answer key | Auto-final (not AI; audited) |
| **Subjective** (Short, Long, Essay, Numerical w/ tolerance, Diagram…) | AI rubric-based grading, candidate only + confidence → teacher verification | AI never final |

Policy per exam type can force manual grading for specific questions.

## 2. Objective grading

```
read answer (from student_answers; OCR-derived accepted text)
→ normalize (case/spaces/synonyms per key spec, digit variants)
→ compare vs key (per-option logic for MSQ: all-correct rule)
→ score + record + audit (no LLM involved)
```
Ambiguous normalization → flagged for review rather than guessed.

## 3. AI subjective grading (rubric-based)

Inputs:
- Question text + official answer + marking rubric (criteria/descriptors)
- Student answer — **verified original text** (or teacher-approved translation)

- Retrieved curriculum context (RAG, approved materials) — optional but stored always when used;
  retrieval is **syllabus-scoped** to the exam's approved coverage ∩ blueprint scope
  ([RAG.md](RAG.md) §2b). The attempt inherits `assessment_period_id` from its exam for
  reporting ([ADR-0017](ADR/ADR-0017-assessment-periods-coverage-exam-sets.md)).

Output (structured):
```json
{
  "criteria_marks": [ {"criterion_id": "c1", "marks": 2, "justification": "...", "confidence": 0.9} ],
  "total_score": 7,
  "overall_confidence": 0.84,
  "explanation": "..."
}
```

Guarantees:
- `criteria_marks` sums to `total_score`; both ≤ rubric totals (validated).
- Every mark is **explainable** (criterion + justification) — no black boxes.
- `confidence` + status; below gate → `NEEDS_REVIEW`.

## 4. Evidence chain for grading

`evidence_snapshots` + inline evidence store: question, official answer, rubric,
retrieved chunks (ids + texts + scores), OCR result + crop refs, translation refs, prompt
version, model/provider/version, raw response ref, candidate score, confidence. Complete
context for the teacher and for post-hoc audits.

## 5. Score storage separation

`grading_results` keeps:
- `candidate.{marks, criteria_marks[], explanation, confidence, provider, model, ...}` (AI)
- `final_marks` + `final_by` + `final_at` (teacher authority)

These are physically distinct fields. No pipeline writes `final_marks` except the
teacher-approval path (domain-enforced).

**Separation of duties:** the answer-key approver of an exam must not be the sole final
verifier of that exam's gradings (AUTH_RBAC §2b); verification assignment excludes the
key approver as sole finalizer.

## 6. Verification-driven scores

- `APPROVED` via teacher accept (bulk-accept allowed for above-threshold items, still a
  deliberate teacher action) or modify/reject.
- Rejected → re-grade or manual fix; history preserved.
- Results engine only reads `final_marks`.

## 7. Grading job orchestration

- Per-answer jobs in `worker-grading` (fan-out by attempt), idempotent by grading_id;
  per-question retries; provider failover per router.
- Progress tracked on the attempt; failure states visible to teacher.

## 8. Metrics & eval

- Live: teacher-agreement/override rates, mean absolute error, confidence calibration,
  per-question pass-rate drift ([OBSERVABILITY.md](OBSERVABILITY.md)).
- Offline: golden grading dataset with human-judged marks; regression-gated
  ([AI_EVALUATION.md](AI_EVALUATION.md)).