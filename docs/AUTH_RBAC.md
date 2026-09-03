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
| `questions:approve` | ✓ | ✓ | ✓ | ✓(own)/principal-approve | ✓(draft) | – |
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

## 4. Tenant isolation

- Every tenant-scoped query includes `school_id` filter guaranteed by repository layer.
- Cross-school IDs in requests → 404 (not 403 — avoid existence leaks).
- Bulk operations validated school-wide; super-admin global ops audited.

## 5. Security notes

- Secrets/JWT keys via env/secret manager; keys rotated; tokens signed with asymmetric key
  (ES256) to allow worker validation without secret.
- Rate limits per identity & IP (login, uploads, AI-heavy endpoints).
- All auth/authorization decisions are audited (`audit_logs`).