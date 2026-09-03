# Authentication & RBAC

## 1. Authentication

- **Issuer:** backend (FastAPI). **Scheme:** JWT access (short, 15–60 min) + refresh
  (rotation, revocable, stored hash). Token claims:
  `sub (user_id)`, `school_ids[]`, `role_codes[]`, `is_platform`, `iat/exp`, `jti`.
- MFA (TOTP) supported for admin roles (config-driven).
- Password policy: min 10 chars, breach-check on set; argon2id hashing.
- Tokens bound to device/user-agent where policy requires (session hash).
- Refresh rotation: old refresh invalid on use → detection alert.

## 2. Authorization model

**RBAC** with permission strings:
`resource:action` (e.g., `questions:approve`, `exams:key:read`, `materials:upload`,
`grading:verify`, `schools:write`, `rag:admin`).

Roles (system-seeded) — see `AUTH_RBAC` permission matrix below. Roles are per-user; a user
can hold multiple roles. Platform vs school scope is driven by `school_ids` (empty =
platform only).

**Assessment-period permissions (ADR-0017):** `assessment_periods:*` (create/read periods),
`coverage:write/read/lock` (record + lock teaching coverage; teachers scoped to their own
subjects/classes), `exam_sets:write/publish/read` (examination sets; Principal/Admin/
Exam-Coordinator manage), `exams:version:create` (authorized users may create a new exam
version after publication — never edit a published one).

**Enforcement:** (1) FastAPI dependency resolves claims → principal;
(2) permission check (RBAC) ; (3) **tenancy scope check** — every object's `school_id` must
be in the principal's `school_ids` unless `is_platform`; repositories embed this as a query
filter (not app-level only).

## 2b. Ownership & scoping (meaning of the "own …" qualifiers)

The permission matrix uses scoping qualifiers ("own class", "own draft", "own exam",
"own subjects"). They are defined by **existing domain relationships** — authorization
scope is derived from them, not from a separate ownership registry:

| Resource | Determines "own" scope |
|---|---|
| Teaching coverage | The teacher is teacher-of-record for the subject (`teachers.subjects`) for the class/section in scope; Admin/Principal are school-wide. |
| Materials | The uploading user (`learning_materials.uploaded_by`); "own draft" = a material the teacher uploaded that is not yet APPROVED. |
| Questions | The creating user while the question is CANDIDATE ("own draft"); APPROVED bank questions are school-scoped, not owner-scoped. |
| Exams / blueprints | The creating user while the exam/blueprint is DRAFT; "own exam" (for `exams:key:read`) = the exam's creator or the subject teacher-of-record; after publication, key access follows the `exams:key:read` role scope. |
| Answer keys | Access via `exams:key:read` plus the exam ownership rule above; every access audited (EXAM_INTEGRITY §4). |
| Classes / sections / students | "own class" = the user is the class teacher (`classes.class_teacher_id`) or an assigned section teacher (`sections.teacher_ids`). |
| Grading / verification | Assignment-based (`teacher_verification_items.assigned_to`); unassigned items are workable by any `grading:verify` holder. |
| Subjects | "own subjects" = subjects listed in `teachers.subjects` for the user's school. |

**Per-school role binding:** role assignments are school-bound — a user's effective
role(s) are evaluated **per accessed school**; `roles.school_id: null` = platform-level
role. A user may therefore hold different roles in different schools (e.g., Admin in
school A, Teacher in school B). The tenancy check (§2 step 3) applies per school;
permissions are evaluated against the role(s) bound to that school.

**Separation of duties (exam integrity):** the answer-key approver for exam X must not
be the sole final verifier of gradings for exam X. MVP enforcement: verification
assignment/auto-assignment excludes the key approver (`exam_answer_keys.approval.by`)
as the sole finalizer — another holder of `grading:verify` must be able to perform the
final verification; the key approver may review but may not alone finalize the grading
result for that exam.

## 3. Permission matrix (MVP)

| Permission | Super Admin | Admin | Principal | Teacher | Exam Coord | Student |
|---|---|---|---|---|---|---|
| `schools:*` | ✓ | – | – | – | – | – |
| `users:write` | ✓ (global) | ✓ (school) | – | – | – | – |
| `users:read` | ✓ | ✓ | ✓ (school) | – | – | – |
| `users:roles:write` | ✓ | ✓ | – | – | – | – |
| `teachers:*` | ✓ | ✓ | ✓ | – | – | – |
| `students:*` | ✓ | ✓ | ✓ | ✓ (own class) | ✓ | – |
| `classes:*`, `enrollments:*` | ✓ | ✓ | ✓ | ✓ (own) | ✓ | – |
| `curriculum:*` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `assessment_periods:write` | ✓ | ✓ | ✓ | – | ✓ (read) | – |
| `assessment_periods:read` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `coverage:write` | ✓ | ✓ | ✓ | ✓ (own subjects) | – | – |
| `coverage:read` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `coverage:lock` | ✓ | ✓ | ✓ | – | – | – |
| `exam_sets:write` | ✓ | ✓ | ✓ | – | ✓ | – |
| `exam_sets:publish` | ✓ | ✓ | ✓ | – | ✓ | – |
| `exam_sets:read` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `materials:upload` | ✓ | ✓ | ✓ | ✓ | – | – |
| `materials:approve` | ✓ | ✓ | ✓ | ✓ (own draft) | – | – |
| `materials:read` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `rag:read` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `rag:admin` | ✓ | – | – | – | – | – |
| `questions:write` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `questions:approve` | ✓ | ✓ | ✓ | ✓(own)/principal-approve | ✓ (own drafts only) | – |
| `exams:write` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `exams:generate` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `exams:publish` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `exams:print` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `exams:key:read` | ✓ | ✓ | ✓ | ✓ (own exam) | ✓ (own) | – |
| `exams:admin` | ✓ | – | ✓ | – | – | – |
| `exams:version:create` | ✓ | ✓ | ✓ | – | ✓ | – |
| `submissions:upload` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `grading:write` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `grading:verify` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `grading:read` | ✓ | ✓ | ✓ | ✓ | ✓ | – |
| `results:read` | ✓ | ✓ | ✓ | ✓ | ✓ (own class) | ✓ (own) |
| `analytics:read` | ✓ | ✓ | ✓ | ✓ (own) | ✓ (own) | ✓ (own) |
| `ai:admin` / `ai:read` | ✓ | – | – | – | – | – |
| `jobs:read` / `jobs:write` | ✓ | ✓ | ✓ | ✓ (own) | ✓ (own) | – |
| `audit:read` | ✓ | ✓ (school) | ✓ (school) | – | – | – |

**Qualifier definitions:** every "own …" qualifier above is defined in §2b (Ownership &
scoping). Exam Coordinator `questions:approve (own drafts only)` = may approve candidate
questions they created themselves; other candidates require
Teacher(owner)/Principal/Admin.

## 4. Tenant isolation

- Every tenant-scoped query includes `school_id` filter guaranteed by repository layer.
- Cross-school IDs in requests → 404 (not 403 — avoid existence leaks).
- Bulk operations validated school-wide; super-admin global ops audited.

## 5. Security notes

- Secrets/JWT keys via env/secret manager; keys rotated; tokens signed with asymmetric key
  (ES256) to allow worker validation without secret.
- Rate limits per identity & IP (login, uploads, AI-heavy endpoints).
- All auth/authorization decisions are audited (`audit_logs`).