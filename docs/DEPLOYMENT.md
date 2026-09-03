# Deployment — Docker Compose (single server) + Windows/WSL2 dev

## 1. Deployment topology (MVP)

Single-server Docker Compose (ADR-0013):

```
nginx (TLS termination, static/asset proxy, rate limit)
 ├── api (uvicorn, replicas=1)
 ├── worker-ingestion | worker-embeddings | worker-generation | worker-ocr
 ├── worker-grading | worker-export | worker-analytics
 └── infra containers: mongo | redis | qdrant | minio | (optional) clamav
Volumes (named, WSL2-native): mongodb-data, qdrant-data, minio-data, redis-data
Secrets: .env (gitignored) → containers; TLS certs by volume.
```

Production host: Linux VPS/on-prem box recommended; the same compose stack runs unchanged.

## 2. Windows 11 + Docker Desktop + WSL2 — dev environment (MANDATORY READING)

Development on this machine (Windows, repo on `G:\`) uses Docker Desktop with a WSL2 engine.

### 2.1 Prerequisites (`scripts/check_env.ps1` will verify)
- Windows 11, Docker Desktop ≥ 4.x with **WSL2 backend** (`wsl --status`), `wsl -l -v`.
- A long-lived distro for data (`docker-desktop-data` auto; prefer a user WSL distro, e.g.
  Ubuntu 22.04 for tools + `docker run` convenience).
- Git for Windows with `core.autocrlf` set to `input` (repo enforces LF via `.gitattributes`).
- Python 3.11+ for local tooling (optional), `uv`/pipenv per developer preference.

### 2.2 CRITICAL: persistent storage must live on the WSL2 filesystem
- **Never bind-mount the `G:` drive filesystem into containers for persistent databases.**
  The `//g/...` 9P-backed mounts are slow, and MongoDB/Qdrant/MinIO rely on mmap/flock and
  sync semantics that are unreliable/severely degraded on 9P.
- Use **named volumes** (created inside the WSL2 VM, stored under `docker-desktop-data`'s
  ext4): `mongodb-data`, `qdrant-data`, `minio-data`, `redis-data`.
- Named volumes are the default in the compose file; host-dir bind mounts are only allowed
  for configuration files (read-only) and are flagged in review.
- Accessing named-volume data from Windows: use `\\wsl$\<distro>\...` or
  `docker run --rm -v <vol>:/data ...` — do not edit database files from Windows tools.

### 2.3 Performance notes
- Keep code on the Windows drive (userspace) but build/run in containers; WSL2 I/O for
  source is acceptable (Docker Desktop uses its own VM for build context — consistent).
- Set Docker Desktop resources: ≥ 4 CPU / 8 GB RAM for dev (MongoDB + Qdrant + MinIO +
  6 workers is heavy); enable "Use the WSL 2 based engine".
- MinIO on WSL2 volumes has known fsync costs; acceptable for dev; for real capacity
  planning see §6.

### 2.4 Line endings & git
- `.gitattributes` enforces LF for text; configure `git config core.autocrlf input` so
  checked-out files are LF for Python/Shell correctness in containers.
- Watch for CRLF in `.sh`/`.py` scripts run by Docker → `sed -i 's/\r$//'` if artifacts.

### 2.5 Firewall & networking
- Services bind on `127.0.0.1` in dev (`ports`) — never `0.0.0.0` for Mongo/Qdrant/MinIO
  outside Docker networks.
- WSL2 localhost forwarding works for `localhost:8080` etc. If not, use the WSL IP.

## 3. Configuration & secrets

- `<service>.env` from `infrastructure/env/*.example`; `MONGODB_URI`, `QDRANT_URL`,
  `REDIS_URL`, `MINIO_*`, `JWT_*`, per-provider API keys live in env/secrets — **never in
  source** ([SECURITY.md](SECURITY.md) §4).
- Provider keys for MVP defaults: prefer local (Ollama) to run keyless in dev; cloud keys
  optional via `ai_provider_*`.

## 4. TLS

- Prod: Let's Encrypt via nginx (`certbot` container or pre-issued certs); HTTP → HTTPS
  redirect; HSTS.
- Dev: self-signed or mkcert for `localhost`; API localhost HTTP allowed by config flag.

## 5. Upgrade procedure (MVP)

1. `docker compose pull` + review migrations (backend runs `schema_migrations`).
2. Maintenance window: stop workers → run migrations → start (`api` first, then workers).
3. Rollback: tagged images + restore from backup if needed; Qdrant re-index on model change
   is alias-safe (QDRANT.md §8).
4. Smoke: health endpoints, one material ingest, one paper grade journey.

## 6. Host sizing guidance (single school MVP)

| Workload | Min | Notes |
|---|---|---|
| dev (Docker Desktop) | 4 vCPU/8 GB | heavy Qdrant+MinIO+6 workers |
| prod school (~300–1000 students, ≤ 4 exam days/term) | 8 vCPU/16 GB, NVMe | OCR + embedding bursts; watch queue depth |
| growth | split workers to a 2nd host | see SCALABILITY.md |

## 7. Health & readiness

- `/healthz` (liveness), `/readyz` (deps: mongo/qdrant/redis/minio) per API container.
- Worker heartbeat + queue depth exposed for Grafana ([OBSERVABILITY.md](OBSERVABILITY.md)).