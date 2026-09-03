# ADR-0003 — MongoDB as System of Record + JWT + RBAC + Tenant Isolation

- **Date:** 2026-09-02
- **Status:** Approved

## Context

Product mandates MongoDB as the primary application database and strong multi-school
tenancy. Security requirements demand role-based access control, and the repository must
guarantee tenant isolation at the data layer.

## Decision

1. **MongoDB = system of record** for all structured application data; Qdrant is derived,
   object storage holds artifacts (see ADR-0005, ADR-0007). This includes the assessment
   calendar and syllabus entities: `assessment_periods`, `curriculum_coverage`, and
   `examination_sets` are tenant-scoped collections with `school_id`-leading indexes
   (ADR-0017).
2. **AuthN:** JWT (ES256) access + rotating refresh tokens; MFA for admins; argon2id
   password hashing.
3. **AuthZ:** RBAC with `resource:action` permissions; role matrix in AUTH_RBAC.md; enforced
   by FastAPI dependencies.
4. **Tenant isolation:** repositories inject mandatory `school_id` filters (from principal
   scope); never caller-supplied; cross-school access → 404. `school_id` leads composite
   indexes on tenant-scoped collections (including the new period/coverage/exam-set
   collections).
5. Optimistic concurrency via `rev` on versioned documents; transactions only where
   multi-document atomicity is required (e.g., publishing an exam version or locking
   coverage for an examination set).

## Consequences

- **Positive:** single logical DB; strong isolation baseline; audit-friendly; simple
  multi-school groundwork.
- **Negative:** No relational joins (mitigated by deliberate embedding/reference design,
  DATABASE.md); requires discipline in repository layer.

## Invalidation triggers

Move to SQL for core transactional data (not anticipated for MVP); requirement for
cross-tenant aggregated analytics without tenant scoping (would need explicit policy + views).