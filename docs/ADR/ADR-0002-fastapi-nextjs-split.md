# ADR-0002 — FastAPI Backend / Next.js Frontend, Physically Separate

- **Date:** 2026-09-02
- **Status:** Approved

## Context

Product requirement: frontend and backend MUST be physically separated; planned stack is
Next.js/React/TypeScript (frontend) and Python/FastAPI (backend). Backend logic must not
live inside the frontend project.

## Decision

1. Two independent applications:
   - `frontend/` — Next.js (App Router), React, TypeScript. Handles UI, client state,
     forms, auth context. Never touches databases/Qdrant/storage directly; consumes only
     `/api/v1`.
   - `backend/` — Python 3.11+, FastAPI, Pydantic v2. All business logic, data access,
     AI, jobs.
2. API contract = OpenAPI generated from FastAPI; frontend mirrors types from it.
3. Deployment: nginx serves static frontend + proxies `/api/*` to backend container.
4. Developer workflow: dev servers for both, `frontend/.env` points to backend URL only.

## Consequences

- **Positive:** independent scaling, security isolation, language-appropriate teams/tools,
  prevents accidental backend-in-frontend logic.
- **Negative:** duplicated validation between TS types and Pydantic (mitigated by contract
  tests + schema sync), two build systems.

## Invalidation triggers

Monorepo platform constraint requiring combined builds (not anticipated); decision to move
frontend to a different framework (still separate).