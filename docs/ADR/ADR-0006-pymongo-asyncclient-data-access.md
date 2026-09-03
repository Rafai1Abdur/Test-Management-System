# ADR-0006 — PyMongo Async API (`AsyncMongoClient`), no Motor

- **Date:** 2026-09-02
- **Status:** Approved
- **Supersedes plan for:** Motor-based data access (rejected prior to implementation;
  no Motor code was ever written — greenfield).

## Context

Greenfield Python/FastAPI project. FastAPI is async; we need async MongoDB access that is
native, maintained by the MongoDB driver team, and current. The MongoDB Python driver now
ships a first-party async API: `pymongo.AsyncMongoClient` (introduced in PyMongo 4.9,
current 4.17.x stable at decision time).

## Decision

1. Use **`pymongo.AsyncMongoClient`** (PyMongo ≥ 4.9, target 4.17.x) for all app code.
2. **Do not adopt Motor** (separate third-party maintainer; redundant now that the native
   async API exists; avoids a legacy dependency).
3. Data-access layer: domain **repositories** over `AsyncMongoClient` + Pydantic v2
   schemas; no ORM/ODM abstraction. Connection pooling/lifecycle owned by `infrastructure/db`.
4. Celery task bodies that need Mongo are async and awaited within the event loop (or use a
   short-lived sync client in dedicated sync helpers — never both on cross domains).

## Consequences

- **Positive:** first-party driver (regular releases, MongoDB version support guarantees);
  async-native in FastAPI (no thread-pool blocking); no Motor deprecation risk; simpler
  dependency tree.
- **Negative:** Motor-like convenience APIs (e.g., some sugar) absent — acceptable; we
  control our own models anyway.

## Invalidation triggers

MongoDB deprecating the async API (not expected); adoption of a different app DB.