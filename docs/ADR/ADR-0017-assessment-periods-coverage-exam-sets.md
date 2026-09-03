# ADR-0017 — Assessment Periods, Teaching Coverage, Examination Sets & Syllabus-Aware Scope

- **Date:** 2026-09-02
- **Status:** Approved (amendment to the documentation package)

## Context

The original architecture modeled "academic year → exam" but lacked an explicit
**assessment calendar** and **syllabus governance**. Schools assess in multiple windows
per year (First/Second/Third Term, Quarter 1–3, Mid-Term, Final Term, custom periods),
each with its own *taught* curriculum. Without these concepts, exam generation cannot
guarantee that questions come only from chapters actually taught and approved, cumulative
exams cannot be modeled correctly, results cannot be reported per period, and multiple
exams of the same period (an "Examination Set") cannot be scheduled/published as a group.

## Decision

1. **New conceptual hierarchy** (period ≠ exam — strictly separate entities):

   ```
   Academic Year → Assessment Period → Teaching/Curriculum Coverage → Examination Set
                → Individual Exam → Exam Blueprint → Exam Version → Exam Paper Instance
   ```

2. **`assessment_periods` collection** — time windows within an academic year
   (TERM | QUARTER | MID_TERM | FINAL_TERM | FINAL_EXAM | CUSTOM), ordered, with
   `PLANNED|ACTIVE|CLOSED` status.

3. **`curriculum_coverage` collection** — teaching coverage per (year, period, subject,
   *optional* class/section). **Chapter-level is mandatory**; section/topic-level is
   optional. Coverage states: `NOT_STARTED | IN_PROGRESS | COMPLETED | EXCLUDED`.

   **Inheritance/resolution:** class/section-specific coverage is used when present;
   otherwise the grade+subject default coverage applies (fallback). Documented as an
   explicit resolution rule.

4. **Coverage locking:** coverage-level state machine `DRAFT → REVIEWED → LOCKED`. An exam
   can require locked coverage depending on school configuration. Once locked for an exam
   scope, changes are audited and must never silently alter created/published artifacts
   (they trigger a new version).

5. **`examination_sets` collection** — groups exams of one assessment period (grade,
   class/section scope, subjects, schedule, publication state). For MVP, **scheduling is
   advisory** (dates do not hard-enforce by default); the data model still allows schools
   to enable hard scheduling windows later.

6. **Exam scope modes** — `NEW_ONLY | CUMULATIVE | SELECTED_CHAPTERS | FULL_SYLLABUS |
   CUSTOM`. **Syllabus lock:** once an exam reaches APPROVED/PUBLISHED, its
   scope/coverage is immutable; post-publication coverage changes require a new exam
   version (never silent mutation).

7. **Scope resolution = materialization + revalidation:**
   - At **blueprint creation**: resolve the requested scope from approved/locked teaching
     coverage and materialize the resolved chapter/section scope into the blueprint
     (teacher reviewable/editable while DRAFT).
   - At **generation**: revalidate the materialized scope against current applicable
     coverage; **fail closed** if invalid; never silently expand.

8. **Generation invariant:**

   ```
   generation_scope ⊆ approved_teaching_coverage ∩ blueprint_scope
   ```

   An exam may only use chapters/sections satisfying **both** the approved examination
   scope and the approved teaching/curriculum coverage. RAG becomes **syllabus-aware**:
   retrieval scope conceptually = Academic Year + Assessment Period + Subject + Grade +
   Approved Teaching Coverage + Selected Chapters/Sections + Approved Learning Materials.

9. **Chapter/section weighting** — optional teacher-defined percentages per chapter;
   tolerance default **±10 percentage points (absolute percentage-point deviation, not
   relative)**, configurable at school/system level. Generated exams validate mark
   distribution against weights within tolerance.

10. **Reporting dimension** — `assessment_period_id` (+ `academic_year_id`) flows into
    student/class/subject results, chapter performance, analytics cubes, and report cards
    (school-branded template with platform-default fallback).

## Consequences

- **Positive:** deterministic, syllabus-governed exam generation; cumulative exams modeled
  cleanly; coverage-aware RAG prevents out-of-syllabus questions; results reportable per
  period; grouped exam scheduling; alignment with exam-integrity versioning.
- **Negative:** added domain complexity (schemas, resolution service, lock semantics);
  teachers must maintain coverage records (mitigated by coaching + optional inheritance);
  additional validation stage at generation.

## Invalidation triggers

Product decision to drop assessment periods/coverage (not anticipated); adoption of an
external SIS as source of truth for coverage would add a sync adapter but not remove the
concepts.