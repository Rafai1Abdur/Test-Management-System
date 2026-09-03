# Risk Register

Each risk: likelihood (L), impact (I), and mitigation. Register reviewed each phase.

| # | Risk | L | I | Mitigation |
|---|---|---|---|---|
| 1 | **Handwriting recognition unreliable** | H | H | Multi-model consensus + confidence gates; teacher verification mandatory below thresholds; never auto-finalize; keep originals; eval datasets measure true rates; user expectations set honestly |
| 2 | **Urdu OCR/handwriting quality (Nastalique)** | H | H | PaddleOCR-UR + VLM consensus; strict gates (**0.60 UR handwriting default**); region-level repair UI; UR eval set is a Phase 2 deliverable; RTL normalization |
| 3 | **Poor image quality (photos of papers)** | M | H | Pre-processing (deskew/perspective/enhance); QR-anchored templates; region-based OCR limiting bad rec; confidence fallback → teacher; per-page repair workflow |
| 4 | **LLM hallucinations (questions/keys/rubrics)** | M | H | Grounded generation (blueprint allowlist + RAG); structural/answer validation; second-model quality pass; human review before exam use; evidence chain exposes sources |
| 5 | **Incorrect answer keys** | M | H | Key approval before publish; answer validation (MCQ exactly-one); publish-time checksum; post-publication correction = new version + audit |
| 6 | **Grading inconsistency (inter-answer/model drift)** | M | H | Rubric-driven criterion grading with justification; frozen prompt+model versions per run; teacher override telemetry; grading eval regressions; calibration monitoring |
| 7 | **RAG retrieval errors (missing/off-topic context)** | H | H | Metadata scoping + authority allowlist; hybrid dense+sparse+RRF+rerank; eval hit@k gates; evidence of retrieved chunks on every artifact; teacher-visible sources |
| 8 | **Provider/model availability (cloud outage)** | M | M | Router eligibility chains + failover; local-first option (Ollama) gives offline mode; retries + DLQ; status dashboard; health checks before selection |
| 9 | **AI cost explosion** | M | M | Cost ledger per call; budgets/alerts/optional hard caps; tiered routing; caching embeddings; local-first school profile (see COST_MANAGEMENT) |
| 10 | **Copyright on uploaded books** | M | M | Upload consent + school license records; material authority classes; admin review; internal-use-only enforcement; takedown workflow (future) |
| 11 | **Student privacy / data protection** | M | H | PII encryption; tenancy isolation tests; data-class policy for AI calls; retention/deletion workflows; consent records; audits; synthetic test data only |
| 12 | **Scaling surprises (queue growth, hot spots)** | M | M | Worker pools per domain; queue-depth alerts; plain page indexes; rollups for analytics; documented scale-out order (SCALABILITY) |
| 13 | **File storage growth / orphan files** | M | M | Bucket lifecycle rules; orphan sweeper; size caps; storage dashboards; archive policy |
| 14 | **Queue/worker failures (crash mid-job)** | M | M | acks_late + idempotent tasks; heartbeats + watchdog requeue; DLQ and alerts; failure-state UX |
| 15 | **Backup loss / restore failure** | L | H | Monthly restore drills; encrypted offsite copies; checksum verification; Qdrant rebuildability as safety net (see BACKUP_RECOVERY) |
| 16 | **Exam leakage / tampering** | M | H | Publication lock; per-paper randomization; encrypted keys + audit; QR/checksums; duplicate-paper detection; education & pre-exam checklist |
| 17 | **Prompt/model regression** | M | M | Prompt versioning; AI Evaluation Lab gates before promotion; shadow runs for new models |
| 18 | **Over-engineering / scope creep** | M | M | This MVP scope; ADR discipline; single-server compose first; extraction only when data shows need |
| 19 | **Adoption friction (teacher trust in AI)** | H | M | Verification-first UX; "why" evidence panel; teacher override statistics visible to leadership; phased rollout with printed-OCR-first |
| 20 | **Stale/wrong teaching coverage → out-of-scope exams** | M | H | Coverage review gate before blueprint; materialized scope revalidated at generation (fail-closed, never auto-expanded); syllabus lock at publish; coverage changes audited; scope-violation metric gated at 0 ([ADR-0017](ADR/ADR-0017-assessment-periods-coverage-exam-sets.md)) |

## Watchlist (updated per phase)
- Urdu handwriting CER trend — gate for Phase 7→8.
- Teacher override rate — if > 30%, threshold recalibration.
- Cloud provider pricing changes — review cost profiles quarterly.