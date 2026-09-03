# Exam Integrity

Exam integrity is a **cross-cutting, first-class domain** (ADR-0015). It protects the exam
lifecycle from drafting through printing, grading, and archival.

## 1. Security goals

- Prevent leakage of live exams, answer keys, and per-paper orderings.
- Detect tampered/duplicate papers and unapproved key access.
- Guarantee that a published exam version is immutable and every correction is a new version.
- Support trustworthy machine-reading of papers (via QR/barcodes).

## 2. Identifiers

| Entity | Identifier | Format |
|---|---|---|
| Exam (workspace) | `exam_id` | ULID (URL-safe) |
| Exam version | `(exam_id, version)` | int sequence |
| Paper instance | `paper_id` | ULID; printed on QR |
| Question | `question_id` | ULID; rendered on page borders/margin |

- All IDs appear on printed pages (machine-readable); QR encodes
  `{exam_id, version, paper_id, page_no, checksum}`.

## 3. Paper instances & randomization

- Publish creates **paper instances** per student (or per printed copy):
  - `order_seed` per paper → per-paper **question ordering** shuffle (seeded, reproducible).
  - Per-paper **MCQ option ordering** shuffle (seeded) — stored per paper, never global.
  - QR/barcode per page; page-level checksums.
- Records in `exam_paper_instances`: seed, question_order[], option_order{}, qr_payload,
  status (`GENERATED → PRINTED → RETURNED → PROCESSING → GRADED → VERIFIED`, `VOID`).
- Reproducibility: orderings reproducible from seed + exam version (deterministic shear).

## 4. Secure answer-key access

- Answer keys stored in `exam_answer_keys`, **encrypted at rest**, RBAC-gated
  (`exams:key:read`). Assigned only to the grading team.
- Every key view/download/print → `exam_access_audit` entry (who/when/what/ip).
- Keys never embedded in student-facing payloads or frontend bundles; download via
  short-lived signed URLs only.

## 5. Publication locking & change control

- Exam `READY → PUBLISHED` is a locking transition: **content, marks, duration,
  instructions, and the examination syllabus/scope (assessment period, coverage mode,
  selected chapters/sections)** are frozen. API rejects edits with `409 EXAM_LOCKED`
  (or `409 SYLLABUS_LOCKED` for scope/coverage changes).
- **Post-publication corrections require a new version** (`PUBLISHED v1` stays immutable;
  `v2` published with change notes + audit). Historical results reference their version.
- Unpublished drafts are versioned too (auditable editing history).

## 5b. Syllabus lock (assessment scope immutability — ADR-0017)

- An exam's scope/coverage is materialized at blueprint creation from approved **locked**
  teaching coverage and frozen at exam `READY` (recorded as `syllabus_snapshot` +
  `scope_snapshot` coverage hash).
- **If teaching coverage changes after an exam reaches APPROVED/PUBLISHED:** the change is
  audited, the existing exam/version is NOT silently mutated; a **new exam version** is
  required (or, for a draft, re-resolution via `resolve-scope`).
- Pass/edit-guardrails: generation revalidates the materialized scope against current
  coverage and **fails closed** (no out-of-scope content, no silent expansion).
- Coverage changes that would invalidate a **published** exam produce an integrity alert to
  exam coordinators (alongside the version-history entry).

## 6. Audit trail & tamper considerations

- Audit events: draft edits, publish, key access, paper generation/regeneration, voids,
  version corrections, return-tracking.
- Tamper detection: paper checksums; duplicate paper_ids (two images with same paper_id →
  alert); unexpected question-order/QR mismatch → item flagged for teacher
  (`TAMPER|AMBIGUOUS` in verification queue).
- Never silently re-grade a tampered item; always surface to teacher.

## 7. Printing controls

- Print jobs logged (item copies, printer/user, time) to correlate returns.
- Answer spaces sized via blueprint (marks × lines) — printed layout must match the
  region templates used for processing.

## 8. Deletion & retention

- Published versions are never hard-deleted (only `ARCHIVED`). Paper-instance rows keep
  links to returned scans.

## 9. API surface summary

See [API.md](API.md) `/exams` and `/papers` groups; integrity-relevant endpoints:
`publish`, `papers (create/regenerate/void)`, `key (view/download)`, `versions`,
`access-audit (logs)`.