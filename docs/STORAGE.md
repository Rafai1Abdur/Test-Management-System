# Object Storage & File Abstraction (STORAGE.md)

## 1. Role

Object storage is the **source artifact store**: books, student papers, generated exams,
reports, OCR crops, evidence raw outputs. Never stored in MongoDB documents; never in
DBFS. MongoDB stores only `file_ref` (storage key + bucket + sha256 + size) and `storage_ref`
pointers.

## 2. Abstraction

```
StorageClient (interface)
 ├── S3StorageDriver        (prod: S3-compatible — MinIO local, AWS/GCP/… cloud)
 ├── LocalDiskDriver        (dev "storage/" — but see WSL2 notes in DEPLOYMENT.md)
 ├── maybe MinioStorageDriver (thin S3 wrapper)
Operations: put_object / get_bytes / presigned_put_url / presigned_get_url /
            head_object / delete_object / list_objects(prefix) / copy_object
```

- All domain code uses the interface; config (`storage.provider`) selects driver.
- Checksums (sha256) computed on write and verified on read where sensitive.
- No business logic branches on driver type.

## 3. Bucket layout

| Bucket | Contents | Access |
|---|---|---|
| `materials/` | uploaded source files (per material version) | staff, RBAC |
| `materials-pages/` | extracted page images, OCR layers | staff |
| `papers/` | submitted scans/photos (original, immutable) | grading staff |
| `papers-crops/` | per-question answer crops (immutable) | grading staff |
| `exams/` | generated PDF/DOCX per version/paper | RBAC + signed URL |
| `keys/` | answer-key exports (encrypted) | RBAC + audit |
| `reports/` | result/analytics report artifacts | RBAC |
| `evidence/` | raw AI outputs, OCR JSON, snapshot bundles | audit role |
| `imports-exports/` | bulk CSV templates/results | staff |
| `quarantine/` | files failing scan | admin only |

Keys include tenant + entity: `materials/{school_id}/{material_id}/{version}/{sha256}.pdf`.

## 4. Presigned URLs & patterns

- Upload: clients request presigned PUT (or stream through API at ≤50 MB). Large files use
  presigned multipart.
- Download: presigned GET, short TTL (5 min default), issued only after RBAC check, logged
  for sensitive artifact classes.
- All URLs are single-use-ish (TTL); no public buckets.

## 5. Dev vs prod

- Dev Local driver stores under `./storage/` — note Windows/WSL2: **bind-mounting the G:
  drive into MinIO is discouraged**; prefer named volumes (ADR-0013, DEPLOYMENT.md).
- Prod: MinIO (self-host) or cloud S3 — same code path (S3 driver).

## 6. Lifecycle & cleanup

- Orphan sweeper job: remove objects with no referencing MongoDB doc (configurable grace).
- Versioned artifacts (exam PDFs per version) never deleted; only `ARCHIVED` flag data.
- Retention follows [DATABASE.md](DATABASE.md) §7.

## 7. Monitoring & backup

- Object counts/sizes per bucket; storage growth dashboard
  ([OBSERVABILITY.md](OBSERVABILITY.md)).
- Backup via MinIO `mc mirror` / cloud bucket replication to encrypted offsite target
  ([BACKUP_RECOVERY.md](BACKUP_RECOVERY.md)).