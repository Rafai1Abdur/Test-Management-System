# Requirements — AI-Powered School Test & Examination Management System

This document consolidates functional and non-functional requirements. Functional
requirements are traced to the 51 product objectives (FR-01 … FR-51) plus the periodic
assessment extension (FR-56 … FR-63). NFRs are lettered (NFR-A …).

**Conventions:** *Must* = MVP/Required. *Should* = Phase 2. *May* = Future/design-for-it.

## Functional requirements (traceability to objectives)

### Identity, tenancy & curriculum (objectives 1–11)
| ID | Objective | Requirement |
|---|---|---|
| FR-01 | Manage schools | CRUD schools, tenant-scoped; super-admin only for cross-school ops |
| FR-02 | Manage teachers | CRUD teachers with user accounts + role assignment |
| FR-03 | Manage administrators | CRUD admins; per-school Admin and platform Super Admin |
| FR-04 | Manage principals | CRUD principals; school oversight permissions |
| FR-05 | Manage students | CRUD students; linked user accounts (optional) |
| FR-06 | Manage classes & sections | Classes with multiple sections per academic year |
| FR-07 | Manage academic years | Academic year entity; all curriculum scoped to a year |
| FR-08 | Manage subjects | Subjects with grade-level binding and language metadata |
| FR-09 | Manage chapters | Chapters under subject; order, code, learning objectives |
| FR-10 | Manage topics & sections | Topics and sections under chapters |
| FR-11 | Manage curriculum | Curriculum = structured hierarchy school/year/subject/class/chapter/topic |

### Materials (objectives 12–15)
| ID | Objective | Requirement |
|---|---|---|
| FR-12 | Upload teaching materials | Authorized roles (Teacher/Principal/Admin/Super Admin) upload via UI/API |
| FR-13 | Multiple file formats | PDF, DOCX, DOC, PPTX, PPT, TXT, Markdown, HTML, EPUB, JPG, JPEG, PNG, WEBP, TIFF — via format adapters (extensible) |
| FR-14 | Extract structure | Pipeline extracts text/structure via content extractors |
| FR-15 | Detect subject/grade/chapter/section/topic | Auto-analysis enriches material metadata; teacher corrects |

### RAG & derived knowledge index (objectives 16–18)
| ID | Objective | Requirement |
|---|---|---|
| FR-16 | Curriculum-aware RAG | Retrieval scoped by school/year/subject/chapter/section/topic; hybrid dense+sparse+rerank |
| FR-17 | Qdrant as vector index | Qdrant = Derived Semantic Knowledge Index; rebuildable from Mongo + storage |
| FR-18 | MongoDB primary | MongoDB is the system of record; Qdrant never primary |

### Question bank & generation (objectives 19–27)
| ID | Objective | Requirement |
|---|---|---|
| FR-19 | Generate tests/exams from selected material | Exam generator scoped to *selected* approved materials only |
| FR-20 | Multiple question types | MCQ, MSQ, True/False, Fill-blank, Short, Long, Essay, Numerical, Matching, Ordering, Diagram, Equation, Table, Custom |
| FR-21 | Exam blueprints | Blueprint = structural spec (year, **assessment period**, **examination set**, subject, grade, materials, **coverage mode**, chapters/sections, **teaching-coverage constraint**, **chapter/section weighting**, marks, duration, types, difficulty, Bloom, learning objectives, language, instructions) |
| FR-22 | Question banks | Reusable bank with metadata, versioning, approval states, usage stats |
| FR-23 | Duplicate detection | Lexical + semantic duplicate detection across bank |
| FR-24 | Answer keys | AI drafts keys; teacher must approve before exams are published |
| FR-25 | Marking rubrics | Rubrics generated with questions; criterion-level |
| FR-26 | Teacher review/approval | No generated question/key/rubric enters an exam without review |
| FR-27 | Printable PDF exams | PDF with branding, layout, answer spaces, page numbers |

### Exams & output (objectives 28–31)
| ID | Objective | Requirement |
|---|---|---|
| FR-28 | Printable DOCX exams | DOCX via canonical model (Phase 2/MVP optional) |
| FR-29 | Answer sheets | Machine-readable answer sheets w/ QR/barcode + question IDs (see EXAM_INTEGRITY) |
| FR-30 | Upload student papers | Multi-page scans/photos; per-student attempt ingestion |
| FR-31 | Process scanned papers | Page detection, perspective/rotation correction, cropping, enhancement, region detection |

### Handwriting, language, translation (objectives 32–38)
| ID | Objective | Requirement |
|---|---|---|
| FR-32 | Recognize handwriting | Multi-model handwriting recognition with confidence; **English + Urdu architecturally supported from day one** |
| FR-33 | EN/UR priority | EN + UR first-class; EN in MVP, UR in Phase 2 — same architecture, extra data/gates |
| FR-34 | Extra languages | Extensible per-language adapters/pipelines |
| FR-35 | Translate answers | Optional translation between supported languages (e.g., Urdu answer → English for grading) |
| FR-36 | Preserve originals | Original student answer always preserved; translation stored separately |
| FR-37 | Auto-grade objective | Deterministic grading of objective questions |
| FR-38 | AI-subjective grading | Rubric-based AI grading with confidence, evidence, explanation |

### Verification, results, analytics (objectives 39–48)
| ID | Objective | Requirement |
|---|---|---|
| FR-39 | Confidence scores | All AI outputs carry confidence |
| FR-40 | Teacher verification | Uncertain/low-confidence results → teacher queue |
| FR-41 | Final teacher marks | Teacher-approved score stored separate from AI candidate |
| FR-42 | Student results | Results computed from final scores |
| FR-43 | Class results | Class aggregates |
| FR-44 | Subject results | Subject aggregates |
| FR-45 | Chapter/topic performance | Per-chapter/topic performance metrics |
| FR-46 | Analytics | Analytical views for student/class/question/teacher/school |
| FR-47 | Audit trails | Immutable audit records for all significant actions |
| FR-48 | Multiple AI providers | AI gateway with model registry, routing, cost capture |

### Platform (objectives 49–51)
| ID | Objective | Requirement |
|---|---|---|
| FR-49 | Free/paid/local/cloud/self-hosted models | Provider adapters cover all hosting modes |
| FR-50 | Scalable | Stateless API, queues, workers, idempotent jobs, indexes, pagination |
| FR-51 | AI jobs scale | Independent worker pools per workload; queue depth monitored |

### Assessment periods, coverage & examination sets (core extension)
| ID | Requirement |
|---|---|
| FR-56 **Assessment periods** | Multiple assessment periods within an academic year (First/Second/Third Term, Quarter 1–3, Mid-Term, Final Term, custom school-defined). Period ≠ exam. Ordered; PLANNED/ACTIVE/CLOSED |
| FR-57 **Teaching/curriculum coverage** | Record taught chapters per (year, period, subject, optional class/section). States `NOT_STARTED\|IN_PROGRESS\|COMPLETED\|EXCLUDED`; chapter-level mandatory, section/topic-level optional; class/section-specific coverage overrides grade+subject default; teachers + authorized admins update |
| FR-58 **Examination sets** | Group exams of a period (grade, class/section scope, subjects, status, publication state, advisory schedule) |
| FR-59 **Examination scope modes** | `NEW_ONLY \| CUMULATIVE \| SELECTED_CHAPTERS \| FULL_SYLLABUS \| CUSTOM`; scope derived from approved teaching coverage |
| FR-60 **Syllabus-aware generation** | **Never** auto-generate questions from chapters outside approved examination scope AND approved teaching coverage (`generation_scope ⊆ approved_teaching_coverage ∩ blueprint_scope`); fail closed |
| FR-61 **Chapter/section weighting** | Optional per-chapter weighting; generated exam validates marks distribution within configurable tolerance (default ±10 percentage points, absolute) |
| FR-62 **Syllabus lock** | Coverage/scope immutable once exam APPROVED/PUBLISHED; post-publication coverage change ⇒ new exam version, never silent mutation |
| FR-63 **Period reporting dimension** | `assessment_period_id` (+ year, examination set) propagates to student/class/subject results, chapter performance, analytics, and report cards |

## Cross-cutting functional requirements
- FR-52 **AI Evidence Chain**: every AI artifact stores evidence (source materials, chunks,
  prompt version, model+version, raw response, confidence, status).
- FR-53 **Exam integrity**: unique exam/paper IDs, QR/barcodes, randomized question/option
  ordering, secure answer keys, publication locking, post-publication change control, audit.
- FR-54 **AI Evaluation Lab**: golden datasets + regression gates for RAG, OCR, handwriting
  (EN/UR), translation, question generation, grading.
- FR-55 **AI never final**: OCR, handwriting, translation, questions, keys, and grading all
  require confidence/status handling; nothing autogenerated is silently authoritative.

## Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-A **Security** | OWASP-aligned; HTTPS; RBAC; tenant isolation on every query; file validation + malware scan on uploads; secrets via env/secret store; encrypted at rest where feasible; student data treated as sensitive |
| NFR-B **Performance** | API p95 < 500 ms for CRUD reads; retrieval p95 < 1 s; grading job intended < 1 min/answer (model dependent); uploads streamed |
| NFR-C **Reliability** | Jobs idempotent with retries; failed jobs produce error states + retryable; RPO ≤ 24 h, RTO ≤ 4 h for MVP |
| NFR-D **Scalability** | Modular monolith that can split; stateless API; worker pools; connection pooling; pagination everywhere |
| NFR-E **Observability** | Structured logs; metrics (API latency, queue depth, OCR/AI failures, token usage, cost); tracing-ready (OpenTelemetry) |
| NFR-F **Auditability** | Append-only audit + evidence chain; tamper-evident where practical (hash chains) |
| NFR-G **Extensibility** | New formats/providers/languages via adapters/registries; no core-module changes |
| NFR-H **Usability** | Teacher-first workflows; verification UI shows evidence and one-click accept |
| NFR-I **Portability** | Runs on Windows + Docker Desktop + WSL2 (dev) and Linux (prod); all state in volumes safe for WSL2 |
| NFR-J **Cost control** | Per-call cost ledger; per-tenant budgets; routing considers cost |
| NFR-K **Compliance** | Copyright care for textbooks; data deletion on request; AI provider data-policy consents |

## Out of scope / explicitly NOT in MVP
- Perfect handwriting recognition (never promised for any language).
- Full neural machine translation quality for low-resource pairs.
- Multi-school platform features beyond tenancy groundwork.
- Kubernetes-based deployment; full microservices topology.