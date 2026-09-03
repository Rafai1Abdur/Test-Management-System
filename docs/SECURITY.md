# Security Architecture

Applies to the whole platform; student education data is **sensitive application data**.

## 1. Threat-model themes

| Theme | Countermeasures |
|---|---|
| Unauthorized access | JWT+RBAC, tenancy filter by repository, MFA for admins, session binding |
| File-borne attacks | Magic-byte + MIME allowlist, extension allowlist, size caps, ClamAV scan, macro/Script stripping (DOCX/PPTX XML sanitize), PDF JS disarming, image re-encode (strip EXIF), EPUB ZIP-bomb limits |
| Data leakage (keys/questions) | RBAC `exams:key:read`, encrypted-at-rest keys, access audits, signed short-lived URLs, no answers in student payloads |
| Tampering & re-scan | paper checksums, duplicate paper detection, QR integrity, audit hash-chain |
| Malicious AI prompts | prompt injection hardening: separator buffer, explicit instruction blocks, output validation, structured schemas, no system-prompt secrets, human review gates |
| AI data exfiltration | per-school data-policy routing (local-first option), PII redaction policy for cloud calls, consent records |
| Credentials/secrets | env/secret-store only; no secrets in repo; rotation; `.env` ignored |
| Denial of service | rate limiting, upload caps, queue backpressure, timeouts on AI calls, auth on all endpoints |

## 2. Transport & crypto

- HTTPS everywhere (TLS 1.2+; auto certs; internal mTLS optional MVP).
- Passwords: argon2id; JWT: ES256; refresh tokens hashed at rest.
- At rest: object storage SSE (MinIO: server-side encryption in prod); MongoDB encryption
  at rest on cloud/prod big clusters; PII fields app-encrypted
  (students: CNIC, DOB, guardian contacts).

## 3. Secure file access

- Upload: presigned PUT (MinIO) or streamed via API w/ validation; quarantine bucket until
  scan passes.
- Download: short-lived presigned GET (5 min default), RBAC-checked at issuance, audited
  for sensitive artifacts (papers, keys, reports).

## 4. Secrets management

- Env-based config + optional HashiCorp Vault/SOPS for prod; docker secrets for Compose.
- Provider keys per-adapter; rotation supported (config reload).

## 5. Rate limiting & abuse

- Per-user/per-IP buckets: login (5/min), upload (10/min), AI-heavy endpoints (queue-based
  anyway), download (30/min). Exceeded → `429` + audit.
- Anomaly signals (burst of same-paper scans) surfaced for admin.

## 6. Auditability

- `audit_logs` append-only with hash-chain (`hash_prev`) — tamper-evident audit trail.
- All auth decisions, key access, material approval, grading decisions, teacher overrides,
  config changes logged.

## 7. Privacy & data deletion

- Data classification: student PII, exam content, AI evidence — separate retention rules
  (see [DATABASE.md](DATABASE.md) §7).
- GDPR-style: export on request; deletion workflows (erase PII from active store, mark
  evidence anonymized) — implemented as documented processes, not ad hoc deletes.
- Minors: guardian/principal consent records for AI processing of student work.

## 8. AI provider data policies

- Per-school config: `cloud_allowed`, `student_answers_cloud`, approved provider list.
- Router enforces hard filters; violations = hard failure (no silent fallback).
- Recording: every call's data-class in `model_runs` → policy reports.

## 9. Dependency security

- `pip-audit`/`npm audit` in CI; Docker base images pinned + scanned; dependencies
  freeze-locked; Trivy scan in pipeline.

## 10. Backup security

Backups encrypted; offsite copy; access restricted + audited (see [BACKUP_RECOVERY.md](BACKUP_RECOVERY.md)).