# Teacher Verification

## 1. Purpose

The verification queue is the **human authority layer**: it is where uncertain or contested
AI results are resolved by teachers, and where final scores are set. It is not optional —
every AI-graded subjective answer passes through it per policy
(ADR-0012, "AI is never the final authority").

## 2. Queue sources

| Reason | Trigger |
|---|---|
| `LOW_CONFIDENCE` | AI grading confidence < task threshold |
| `OCR_AMBIGUOUS` | OCR/handwriting confidence < threshold, or strong model disagreement |
| `TRANSLATION_AMBIGUOUS` | translation low confidence |
| `FLAGGED` | teacher/manual flags |
| `TAMPER` | integrity checks (duplicate paper_id, checksum mismatch) |
| `REVIEW_REQUIRED` | question type/strategy requires manual review |

Items live in `teacher_verification_items`; each references one `grading_results` row.

## 3. Review workspace — full data contract

For each item the teacher sees (GET `/api/v1/verifications/{id}`):

| Block | Content |
|---|---|
| Paper context | original scan (page/crop), attempt meta |
| Answer evidence | crop image + raw OCR output + language + translation (if any) |
| Question | full question text |
| Reference | official answer + rubric (with criterion marks) |
| AI interpretation | candidate marks, per-criterion justification, confidence, explanation |
| Grounding | retrieved curriculum chunks (evidence chain) |
| System info | provider, model, prompt version (via evidence) |

## 4. Teacher actions

| Action | Effect |
|---|---|
| **Accept** | candidate score becomes final (bulk-accept supported for above-threshold items) |
| **Modify score** | teacher sets final per-criterion marks (records delta vs candidate) |
| **Correct OCR/answer** | teacher edits the recognized answer; corrected row references original (immutable) |
| **Correct interpretation** | teacher fixes language/translation on record |
| **Reject** | item returns for re-processing or manual grading |

Every action: audit event + `final_marks/final_by/final_at` stored on grading_result;
candidate values never mutated.

## 5. Worklists & assignment

- Filters: reason, question type, class, exam, subject, confidence range, assigned-to.
- Assignment to teachers (manual or round-robin); per-teacher workload view.
- Notifications on assignment/completion ([OBSERVABILITY.md](OBSERVABILITY.md)).

## 6. Policy configuration

- Thresholds per task/language (school-overridable) — see AI_ARCHITECTURE §2.2.
- "Require verification for ALL subjective answers" option (strict schools) — default **on**
  for MVP.

## 7. UI/UX principles

- Teacher-first workflows: no raw DB fields; the card shows answer-to-justification chain
  in one screen; keyboard-friendly accept/reject/edit; batch screen for bulk-accept with
  filters to exclude ambiguous ones.
- Mobile-friendly crop zoom; RTL text rendering for Urdu.

## 8. Metrics

- Time-to-review, item counts by reason, override rate, mean absolute delta
  (candidate vs final) — live telemetry in [OBSERVABILITY.md](OBSERVABILITY.md);
  offline in eval regression gates.