# API Design — /api/v1

REST/JSON over HTTPS. OpenAPI generated from FastAPI (backend) is the contract source of
truth; this document is the authoring spec. Frontend consumes only this surface.

## 1. Conventions

**Base path:** `/api/v1`. **Auth:** `Authorization: Bearer <JWT>` (see
[AUTH_RBAC.md](AUTH_RBAC.md)); platform + school scoping from token.

**Errors:** uniform envelope:
```json
{ "error": { "code": "VALIDATION_ERROR", "message": "...", "details": {...}, "trace_id": "..." } }
```
HTTP semantics: `400` bad request, `401` unauthenticated, `403` forbidden, `404` not found,
`409` conflict (version/lock), `422` validation, `429` rate-limited, `500` internal,
`503` unavailable (queue/dependency down).

**Pagination:** cursor-based for list endpoints with high churn (`?cursor=&limit=`, default
50, max 200); `X-Next-Cursor` header + `items[]` body for others. Offset allowed for small
admin lists.

**Filter/Sort/Select:** `filter=` JSON (`{"school_id":"..","status":"APPROVED"}`),
`sort=field:asc` (allowed sort keys per endpoint), projection via `fields=`.

**Idempotency:** mutating `POST`s accept `Idempotency-Key` (UUID). Duplicate key within TTL
(24 h) returns original response (stored fingerprint). Safe for uploads, paper generation,
grading triggers.

**Job endpoints:** create-and-poll pattern — mutating heavy endpoints return
`202 {job_id, status}` unless synchronous (CRUD). Jobs pollable via
`GET /jobs/{id}` and events via webhook/poll.

## 2. Endpoint groups

### auth
| Endpoint | Method | Purpose | Authz |
|---|---|---|---|
| `/auth/login` | POST | email+password → access/refresh tokens | public |
| `/auth/refresh` | POST | rotate refresh token | bearer |
| `/auth/logout` | POST | revoke refresh | bearer |
| `/auth/me` | GET | current user + roles + schools | bearer |
| `/auth/change-password` | POST | password change | bearer |

### users
| `/users` | POST | create user (invites user) | `users:write` |
| `/users` | GET | list (tenant/global) | `users:read` |
| `/users/{id}` | GET/PATCH/DELETE | manage | `users:write` |
| `/users/{id}/roles` | PUT | assign roles | `users:roles:write` |

### schools (super-admin)
| `/schools` | POST/GET | platform tenants | `schools:write/read` |
| `/schools/{id}` | GET/PATCH | manage | `schools:write` |
| `/schools/{id}/config` | GET/PATCH | per-school config (thresholds, branding, policies) | `schools:write` |

### teachers / students / classes / subjects / chapters / topics / enrollments
| `/teachers` | POST/GET | create/list (school-scoped) | `teachers:write/read` |
| `/teachers/{id}` | GET/PATCH/DELETE | manage | `teachers:write` |
| `/students` | POST/GET | create (batch CSV accepted) / list | `students:write/read` |
| `/students/{id}` | GET/PATCH | manage | `students:write` |
| `/classes` | POST/GET | classes | `classes:write/read` |
| `/classes/{id}/sections` | GET/POST | sections | `classes:write/read` |
| `/academic-years` | GET/POST/PATCH | school years | `curriculum:write` |
| `/subjects` | GET/POST/PATCH | subjects | `curriculum:write/read` |
| `/chapters` | GET/POST/PATCH | chapters | `curriculum:write/read` |
| `/topics` | GET/POST/PATCH | topics | `curriculum:write/read` |

### assessment periods / curriculum coverage / examination sets
| `/assessment-periods` | GET/POST/PATCH | periods within year (TERM/QUARTER/MID_TERM/FINAL_TERM/FINAL_EXAM/CUSTOM) | `assessment_periods:write/read` |
| `/academic-years/{id}/assessment-periods` | GET | periods for a year (convenience) | `assessment_periods:read` |
| `/curriculum-coverage` | GET/POST | coverage list / create (period, subject, class/section optional) | `coverage:write`/`coverage:read` |
| `/curriculum-coverage/{id}` | GET/PATCH | detail / bulk chapter+section status update | `coverage:write`/`coverage:read` |
| `/curriculum-coverage/{id}/lock` | POST | DRAFT → REVIEWED → LOCKED (coverage lock) | `coverage:lock` |
| `/examination-sets` | POST/GET | create/list (filters: period, grade, status) | `exam_sets:write/read` |
| `/examination-sets/{id}` | GET/PATCH/DELETE | manage (schedule advisory in MVP) | `exam_sets:write` |
| `/examination-sets/{id}/exams` | GET | list member exams | `exams:read` |
| `/examination-sets/{id}/publish` | POST | set → PUBLISHED (does not auto-publish member exams) | `exam_sets:publish` |
| `/examination-sets/{id}/report-card` | GET | report card per set (PDF via export worker) | `results:read` |

### knowledge / rag
| `/knowledge/search` | POST | scoped retrieval (school/year/**assessment period/coverage-aware scope** filters; returns chunks + scores) | `rag:read` (teacher+) |
| `/knowledge/rebuild` | POST | enqueue re-index (alias-safe) | `rag:admin` |
| `/knowledge/collections` | GET | collection/version status | `rag:admin` |

### questions & question-bank
| `/questions` | GET | bank list (filters: type/subject/chapter/status/...) | `questions:read` |
| `/questions` | POST | manual create / import | `questions:write` |
| `/questions/{id}` | GET | full detail (answers RBAC-gated) | `questions:read` / `questions:key:read` |
| `/questions/{id}` | PATCH | edit (rev++ → version) | `questions:write` |
| `/questions/{id}/approve` | POST | approve candidate | `questions:approve` |
| `/questions/{id}/reject` | POST | reject | `questions:approve` |
| `/questions/{id}/versions` | GET | history | `questions:read` |
| `/questions/aggregate` | GET | stats (usage, p-value, dupes) | `questions:read` |

### exam blueprints / exams / papers / keys
| `/exam-blueprints` | POST/GET | create/list (schema includes assessment_period_id, examination_set_id, scope mode, weighting + tolerance) | `exams:write/read` |
| `/exam-blueprints/{id}` | GET/PATCH/DELETE | manage (editable while blueprint DRAFT; generation-semantics edits affecting a derived exam beyond DRAFT require a new blueprint revision) | `exams:write` |
| `/exam-blueprints/{id}/resolve-scope` | POST | **materialize scope** from approved/locked coverage → review | `exams:write` |
| `/exam-blueprints/{id}/generate` | POST | run generation (revalidates scope; fail closed) → `202 {job_id}` | `exams:generate` |
| `/exams` | POST/GET | create draft / list (gains assessment_period_id, examination_set_id, blueprint_id; syllabus frozen at READY) | `exams:write/read` |
| `/exams/{id}` | GET/PATCH | manage draft | `exams:write` |
| `/exams/{id}/validate` | GET | pre-publish validation report | `exams:write` |
| `/exams/{id}/publish` | POST | lock + version snapshot | `exams:publish` |
| `/exams/{id}/versions` | GET | list immutable versions | `exams:read` |
| `/exams/{id}/print` | POST | queue PDF/DOCX + paper instances | `exams:print` |
| `/exams/{id}/key` | GET | signed key download (audited) | `exams:key:read` |
| `/exams/{id}/key/access-log` | GET | access audit | `exams:admin` |
| `/exams/{id}/papers` | POST | generate paper instances (per student) | `exams:publish`/`exams:print` |
| `/exams/{id}/papers` | GET | list instances | `exams:read` |
| `/exams/{id}/papers/{paper_id}` | GET | detail (QR, seed, status) | `exams:read` |
| `/exams/{id}/papers/{paper_id}/void` | POST | void a paper | `exams:admin` |

### submissions (student papers)
| `/submissions` | POST | multipart (exam_id/paper_id, student, images/pdfs) → `202 {attempt_id, job_id}` | `submissions:upload` |
| `/submissions` | GET | list attempts (filters) | `submissions:read` |
| `/submissions/{attempt_id}` | GET | detail + progress | `submissions:read` |
| `/submissions/{attempt_id}/retry` | POST | re-enqueue processing | `submissions:write` |
| `/submissions/{attempt_id}/answers` | GET | reconstructed answers + OCR + translations | `submissions:read` |

### ocr / translation / grading / verifications / results
| `/ocr/regions/{id}` | GET | region detail (crop, engines, confidence) | `grading:read` |
| `/translation/answers/{id}` | GET | translation detail | `grading:read` |
| `/grading/attempts/{attempt_id}` | POST | enqueue grading | `grading:write` |
| `/grading/results/{id}` | GET | result + evidence chain | `grading:read` |
| `/verifications` | GET | queue list | `grading:verify` |
| `/verifications/{id}` | GET | full review bundle (see TEACHER_VERIFICATION) | `grading:verify` |
| `/verifications/{id}/accept` | POST | accept (optional bulk ids) | `grading:verify` |
| `/verifications/{id}/modify` | POST | set final marks | `grading:verify` |
| `/verifications/{id}/correct-ocr` | POST | teacher-corrected text | `grading:verify` |
| `/verifications/{id}/reject` | POST | reject | `grading:verify` |
| `/results/students` | GET | student result views (filters incl. assessment_period_id, examination_set_id) | `results:read` |
| `/results/classes` | GET | class aggregates (period filters) | `results:read` |
| `/results/subjects` | GET | subject aggregates (period filters) | `results:read` |
| `/results/exams/{exam_id}` | GET | exam result summary | `results:read` |
| `/results/report-cards` | GET | report card per (student, period, exam set); school-branded template; grades from `grading_scales` | `results:read` |

### analytics / ai / models / jobs
| `/analytics/student/{id}` | GET | performance, strengths/weaknesses (period filter) | `analytics:read` |
| `/analytics/class/{id}` | GET | class stats, chapter mastery (period filter) | `analytics:read` |
| `/analytics/question/{id}` | GET | p-value, discrimination | `analytics:read` |
| `/analytics/dashboard` | GET | school trends (period dimension) | `analytics:read` |
| `/ai/models` | GET/POST/PATCH | model registry (admin) | `ai:admin` |
| `/ai/runs` | GET | model run log (filters) | `ai:read` (admin) |
| `/ai/usage` | GET | cost ledger reports | `ai:read` (admin) |
| `/ai/datasets` | GET/POST | eval datasets | `ai:admin` |
| `/ai/evals/runs` | GET/POST | run/gate eval | `ai:admin` |
| `/jobs` | GET | list jobs (filters) | `jobs:read` |
| `/jobs/{id}` | GET | status/progress/error | `jobs:read` |
| `/jobs/{id}/retry` | POST | retry failed job | `jobs:write` |
| `/enrollments` | POST/bulk | enroll students | `enrollments:write` |
| `/enrollments` | GET | list filters | `enrollments:read` |

### materials & documents
| `/materials` | POST | multipart upload → `202 {material_id, job_id}` | `materials:upload` |
| `/materials` | GET | list (status/authority/type filters) | `materials:read` |
| `/materials/{id}` | GET/PATCH | detail/update metadata | `materials:write` |
| `/materials/{id}/analysis` | GET | auto-analysis result (candidate) | `materials:read` |
| `/materials/{id}/approve` | POST | approve (→ APPROVED) | `materials:approve` |
| `/materials/{id}/archive` | POST | archive | `materials:write` |
| `/materials/{id}/file` | GET | signed URL download | `materials:read` |
| `/documents/{id}/chunks` | GET | chunk list (for review/eval) | `materials:read` |