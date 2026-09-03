# Backup & Recovery

## 1. Sources of truth to protect

| System | Data on disk (container) | Backup mechanism |
|---|---|---|
| MongoDB | `mongodb-data` volume | `mongodump`/`mongorestore` (also `mongosh` files) + oplog tailing (point-in-time) if replica set later |
| Qdrant | `qdrant-data` volume | Qdrant **snapshot API** (consistent) — but index is **rebuildable**; snapshot = fast-recovery aid |
| MinIO | `minio-data` volume | `mc mirror` (or cloud replication) to encrypted target |
| Config/secrets | `.env`, certs | encrypted vault backup; keys exported to secure storage |
| Code | git | repository backup/mirror (tagged releases) |

## 2. Rebuildability as the safety net

Qdrant is derived (ADR-0007, QDRANT.md). If Qdrant snapshot is unusable/missing:
`replay: rag_chunks → embed → index` (alias swap). RPO impact: candidate only if MongoDB +
MinIO backups are intact. MongoDB is the true SoR; verify Mongo backup **first** in any
disaster.

## 3. Schedule & retention (MVP)

| Item | Frequency | Retention |
|---|---|---|
| Mongo full dump | daily | 14 daily + 4 weekly + 6 monthly |
| Mongo oplog tail | continuous (if replica set) | 24 h PITR window |
| Qdrant snapshot | daily (only if cheap) or on re-index | 3 snapshots |
| MinIO mirror | nightly | 14 day retention; immutable via object-lock where used |
| Config/secrets | on change | 90 days |
| Restore drill | monthly automated | record outcome |

Targets: **RPO ≤ 24 h** (with dump), **RTO ≤ 4 h** (restore + rebuild-in-parallel).

## 4. Backup security

- Backups encrypted (age/openssl or provider KMS); offsite copy; restore access restricted
  and audited; backup integrity checksums verified on write & at drill.

## 5. WSL2/dev specifics

- Named volumes live inside the WSL2 VM. On Windows, back them up by exporting volumes
  (`docker run --rm -v <vol>:/data -v %CD%:/backup alpine tar czf /backup/<vol>.tgz -C /data .`)
  — not by copying `\\wsl$` dirs while containers run.
- Qdrant/MinIO hot-copy of datadir is unsafe; snapshot API / `mc mirror` are the supported
  paths.

## 6. Point-in-time & multi-node note

- MVP single node: dumps only → max 24 h RPO. Adding a replica set (decision in Phase 11)
  provides PITR; document during Phase 11 hardening.

## 7. Restore procedure (documented; drilled)

1. Halt writes or spin staging compose on same images.
2. Restore Mongo dump → run migrations → health checks (catalog counts).
3. Restore MinIO mirror; verify checksums on critical artifacts.
4. Restore/rebuild Qdrant (snapshot restore or replay from `rag_chunks`).
5. Re-verify evidence chain refs (spot) + run sanity E2E (upload→grade flow).

## 8. Runbooks index

See `scripts/backup/` and `scripts/restore/` (to be created in Phase 1) + this document;
monthly drill keeps them honest.