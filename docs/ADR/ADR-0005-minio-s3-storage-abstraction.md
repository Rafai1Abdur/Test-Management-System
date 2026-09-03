# ADR-0005 — S3-Compatible Object Storage Abstraction

- **Date:** 2026-09-02
- **Status:** Approved

## Context

Large files (books, papers, generated exams, evidence) must not live in MongoDB; product
requires S3-compatible storage with local development support (MinIO in Docker Compose).

## Decision

1. A `StorageClient` interface with drivers: `S3StorageDriver` (prod: MinIO or cloud S3)
   and `LocalDiskDriver` (dev). All domain code uses the interface (STORAGE.md §2).
2. Bucket/key scheme encodes tenant + entity (e.g. `papers/{school}/{attempt}/{page}.png`);
   checksums on writes; presigned short-TTL URLs for upload/download with RBAC at issuance;
   no public buckets.
3. MongoDB stores only file refs/metadata; never blobs.
4. **Named volumes (WSL2-native) for MinIO dev storage — never bind-mounts into a Windows/G:
   drive DB filesystem** (see ADR-0013).

## Consequences

- **Positive:** storage provider swappable; compliance-friendly; replication/backup reuse;
  upload streaming/presign patterns established early.
- **Negative:** interface seams to maintain; S3 semantics (eventual consistency nuance) have
  to be respected.

## Invalidation triggers

A hard requirement for a non-S3 storage backend and no usable adapter; would add a driver.