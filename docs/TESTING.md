# Testing Strategy

Quality is enforced at multiple levels; AI-heavy components are tested via golden datasets
and regression gates ([AI_EVALUATION.md](AI_EVALUATION.md)).

## 1. Test pyramid

| Level | Scope | Tools (MVP) | Where |
|---|---|---|---|
| Unit | domain logic, repositories (mocked), validation, pipeline components | pytest (pytest-asyncio), freezegun | `backend/tests/unit` |
| Integration | Mongo/Qdrant/MinIO/Redis against real containers (testcontainers or compose dev) | pytest + fixtures | `backend/tests/integration` |
| API/contract | endpoint behavior (authz, tenancy, schemas, idempotency) | httpx ASGI client; OpenAPI schema tests | `backend/tests/api` |
| Frontend | components/features | Vitest + React Testing Library | `frontend/src/**/*.test.ts(x)` |
| E2E | full workflow journeys (UI → API → workers → DB) | Playwright | `tests/e2e` |
| Eval | RAG/OCR/HW/translation/QG/grading on golden sets | eval harness (see AI_EVALUATION) | `evals/` |
| Security | authz matrix, tenant leaks, file validation, injection, headers | pytest + scan tooling (ZAP in CI), Trivy, pip/npm audit | `backend/tests/security` |
| Performance/Load | latency, throughput, queue behavior | locust/k6 + pytest markers | `tests/perf` |
| Failure/Recovery | job crash, provider outage, restore drills | scripts + tests | `tests/failure` |

## 2. Golden datasets (AI Evaluation Lab)

Per capability (`evals/datasets/<capability>/<set>/`), versioned, with labels:
- `rag/` — query → expected chunk/topic hits (hit@k, MRR, nDCG).
- `ocr/` and `hw_en/`, `hw_ur/` — images → expected text (CER/WER).
- `translation/` — EN↔UR pairs (semantic scores + human refs).
- `question_generation/` — blueprint → expected curriculum coverage/correctness labels.
- `grading/` — question+answer+rubric → human-judged marks/edge cases.
Datasets are versioned (`eval_datasets`), CI-runnable, and regression-gated: PRs changing
prompts/models/pipelines must not regress gated metrics ([AI_EVALUATION.md](AI_EVALUATION.md)).

## 3. Non-AI test emphasis

- **Tenancy tests**: every list/detail endpoint tested cross-school → 404.
- **Authz matrix test**: parameterized role×permission over endpoint table.
- **Idempotency tests**: duplicate Idempotency-Key returns same result; duplicate job
  dispatch no double effects.
- **Integrity tests**: publish locks; post-publish corrections require version bump; key
  access audited; paper duplicate detection; checksum mismatch flow; **syllabus lock**
  (scope immutable at READY (syllabus freeze)/PUBLISHED; post-publication coverage change
  ⇒ new version).
- **State machine tests**: material/question/exam/grading lifecycle transitions incl.
  invalid transitions; **assessment period / examination set / coverage lifecycles**
  (DRAFT→REVIEWED→LOCKED coverage; set SCHEDULED→PUBLISHED).
- **Evidence chain tests**: every AI artifact has non-null evidence refs + immutable
  snapshots; candidate-vs-final separation enforced.
- **Coverage-eligibility tests (negative)**: a chapter that is `IN_PROGRESS`,
  `NOT_STARTED`, or `EXCLUDED` produces **zero** generated candidates — even when
  explicitly selected in the blueprint (fail-closed; EXAM_ENGINE §2b, AI_EVALUATION §2).

## 4. CI pipeline (planned)

```
lint+typecheck (ruff, mypy, eslint) → unit → integration (containers)
→ API → eval gates (if AI-affecting change) → build images (Trivy scan)
→ deploy to staging → smoke + Playwright E2E → perf smoke
```
Merge blocked on: unit/integration green; eval gates green for AI-affecting PRs;
no dependency advisories introduced.

## 5. Test data & isolation

- Tenants: randomized school fixtures; PII synthetic only.
- Golden images: hand-generated + public-domain samples (no real student data).
- Eval results stored in `eval_runs` for comparison & drift detection.