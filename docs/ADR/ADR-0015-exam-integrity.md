# ADR-0015 — Exam Integrity as a First-Class Domain

- **Date:** 2026-09-02
- **Status:** Approved

## Context

Exams are high-stakes. Without integrity controls, question order sharing, answer-key
leaks, tampered papers, and post-publication edits undermine everything. Machine-readable
printing (QR) also underpins reliable OCR processing.

## Decision

1. **Exam integrity is a first-class capability** (EXAM_INTEGRITY.md), backed by
   collections `exam_paper_instances`, `exam_answer_keys`, `exam_access_audit`, and
   immutable `exam_versions`.
2. Identifiers: ULID `exam_id`, `(exam_id, version)`, `paper_id`; QR payload
   `{exam_id, version, paper_id, page_no, checksum}`.
3. **Randomization:** per-paper seeded question ordering and MCQ option ordering (stored per
   paper; reproducible from seed); no global fixed order.
4. **Key security:** encrypted at rest, RBAC (`exams:key:read`), every access audited,
   never in student payloads, signed short-lived downloads.
5. **Publication locking:** PUBLISHED is immutable; post-publication corrections → new
   version + change notes; historical results reference their version.
6. **Tamper handling:** paper checksum + duplicate-paper detection; integrity anomalies →
   flagged in verification queue (never silently re-graded).
7. **Syllabus lock (ADR-0017):** the examination scope/coverage (assessment period,
   coverage mode, selected chapters/sections, teaching-coverage-derived syllabus) is frozen
   at `READY` (syllabus freeze; fully locked at `PUBLISHED` — the exam lifecycle has no
   `APPROVED` state). Post-publication coverage changes require a **new exam
   version** — never silent mutation of the published version. Coverage changes affecting a
   locked scope are audited and surface version-history entries.

## Consequences

- **Positive:** trustworthy exams; automated processing synergy (QR→regions); audit trail;
   leak-mitigation.
- **Negative:** workflow strictness (corrections need version bumps); per-paper data volume
   (bounded).

## Invalidation triggers

Product decision to relax publication locking (requires ownership sign-off); per-paper
printing not required for all exams (scope config, not architecture change — open question
#6).