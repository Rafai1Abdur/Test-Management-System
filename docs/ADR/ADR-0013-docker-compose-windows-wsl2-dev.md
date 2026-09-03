# ADR-0013 — Docker Compose Single-Server + Windows 11 / Docker Desktop / WSL2 Dev Rules

- **Date:** 2026-09-02
- **Status:** Approved

## Context

First production target was confirmed as a single server with Docker Compose (no K8s, no
microservices for MVP). Development happens on Windows 11 with the repository on a `G:`
drive and Docker Desktop using the WSL2 backend. Persistent database storage on Windows
bind-mounts (9P) is a known reliability/performance trap for MongoDB/Qdrant/MinIO.

## Decision

1. **Deployment: Docker Compose single server** (nginx + api + worker pools + mongo +
   redis + qdrant + minio; ADR-0001). Linux VPS/on-prem recommended as prod host.
2. **Windows + WSL2 dev rules (normative):**
   - Persistent data uses **named volumes** created inside the WSL2 VM
     (`mongodb-data`, `qdrant-data`, `minio-data`, `redis-data`) — **never bind-mounts
     into the Windows `G:` filesystem for DB datadirs** (mmap/flock/sync on 9P are
     unreliable/slow).
   - Bind-mounts allowed only for read-only config files; flagged in review.
   - Data access via `\\wsl$\...` / `docker run --rm -v` — never edit DB files from Windows
     tools.
   - `.gitattributes` enforces LF; `core.autocrlf input`; guard against CRLF in `.sh`/`.py`
     executed by containers.
   - Services bind to `127.0.0.1` in dev (never `0.0.0.0` for DBs); Docker Desktop
     resources ≥ 4 CPU / 8 GB for dev.
3. Backup/restore for WSL2 volumes: export via `docker run --rm -v` tar, not host copies
   (BACKUP_RECOVERY.md §5).
4. Upgrade/rollback + sizing guidance documented in DEPLOYMENT.md.

## Consequences

- **Positive:** matches product constraint; correct-for-Windows dev; single-server ops;
  clean migration path to scalable deployments later.
- **Negative:** dev perf is heavier; volume management has WSL2 specifics (documented).
- **Positive (ops):** everything containerized; backup/restore drillable.

## Invalidation triggers

Scale requirements beyond a single server (SCALABILITY.md path); move to managed cloud DBs
(atlas/qdrant-cloud) — compose topology adjusts while core ADRs stay.