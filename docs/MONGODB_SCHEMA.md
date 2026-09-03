# MongoDB Schema — Full Collection Catalog

Companion to [DATABASE.md](DATABASE.md). Each collection documents purpose, key fields,
types, relationships, indexes, validation, and lifecycle.

**Conventions:** `_id` ObjectId unless noted; `ref.<c>` = reference; `school_id` mandatory
on all tenant-scoped docs; `created_at`/`updated_at` (UTC) everywhere; public IDs are ULID
strings (`public_id`); `rev:int` = optimistic-concurrency revision on versioned docs.

---

## Identity

### `schools`
**Purpose:** Platform tenants.
**Fields:** `name`, `code` (unique), `address`, `contact`, `config: object` (language
defaults, numbering, watermark), `owner_user_id`, `status: ACTIVE|SUSPENDED`, timestamps.
**Indexes:** unique `code`, `name`.
**Validation/lifecycle:** Super-admin managed. Deactivation only via compliance workflow.

### `users`
**Purpose:** Login principals.
**Fields:** `email` (unique), `phone?`, `password_hash`, `password_salt`,
`password_version:int`, `first_name`, `last_name`, `name_alias?` (Urdu), `locale`,
`roles:ObjectId[] ref.roles`, `school_ids:ObjectId[]` (empty = platform-only),
`status: ACTIVE|DISABLED|PENDING`, `mfa_enabled`, `last_login_at`, `rev:int`, timestamps.
**Indexes:** unique `email`; `school_ids`; `status`.
**Lifecycle:** invite → PENDING → ACTIVE → DISABLED; force password change on first login.

### `roles`
**Purpose:** Role definitions (seed).
**Fields:** `code` unique (`SUPER_ADMIN|ADMIN|PRINCIPAL|TEACHER|EXAM_COORDINATOR|STUDENT`),
`name`, `permissions:string[]`, `is_system`, `school_id:null|ObjectId`, timestamps.
**Indexes:** unique `code`.
**Lifecycle:** system roles immutable; tenant-defined roles supported later.
**School binding:** user role assignments are school-bound — a user's effective role is
evaluated per accessed school (`roles.school_id: null` = platform-level role), so a user
may hold different roles in different schools (AUTH_RBAC §2b).

### `permissions`
**Purpose:** Permission catalog (declarative).
**Fields:** `code` unique (e.g. `questions:approve`), `group`, `description`, timestamps.
**Indexes:** unique `code`.

### `teachers`
**Purpose:** Teacher profiles (1:1 user).
**Fields:** `school_id`, `user_id:ref.users`, `employee_code`, `subjects:ObjectId[]
ref.subjects`, `classes[] ref.classes`, `specialization`, `is_active`, timestamps.
**Indexes:** unique `(school_id, employee_code)`; `user_id`.

### `students`
**Purpose:** Student profiles (sensitive PII).
**Fields:** `school_id`, `user_id:null|ref.users`, `enrollment_no`, `first_name`,
`last_name`, `father_name`, `cnic_bform?` (encrypted), `dob`, `guardian_phone`
(encrypted), `emergency_contact` (encrypted), `language_hints: primary:UR|EN`,
`status: ACTIVE|TRANSFERRED|GRADUATED`, timestamps.
**Indexes:** unique `(school_id, enrollment_no)`; `status`.
**Security:** PII encrypted; see [SECURITY.md](SECURITY.md). Never user-hard-deleted.

---

## Enrollment & curriculum

### `academic_years`
**Purpose:** Year scoping for curriculum/exams.
**Fields:** `school_id`, `name` (`2026-27`), `start_date`, `end_date`,
`status: PLANNED|ACTIVE|CLOSED`, timestamps.
**Indexes:** unique `(school_id, name)`; `(school_id, status)`.

### `assessment_periods`
**Purpose:** Assessment time windows within an academic year (period ≠ exam; ADR-0017).
**Fields:** `school_id`, `academic_year_id`, `name` (`First Term`, `Quarter 1`, `Mid-Term`,
`Final Term`…), `code`, `type` (`TERM|QUARTER|MID_TERM|FINAL_TERM|FINAL_EXAM|CUSTOM`),
`order:int`, `start_date`, `end_date`, `status: PLANNED|ACTIVE|CLOSED`, timestamps.
**Indexes:** unique `(school_id, academic_year_id, code)`; `(school_id, academic_year_id,
order)`; `(school_id, status)`.
**Lifecycle:** PLANNED → ACTIVE → CLOSED; closed periods are read-only for new exam sets.

### `curriculum_coverage`
**Purpose:** Teaching/curriculum coverage per (year, period, subject, optional
class/section). Chapter-level mandatory; section/topic-level optional (ADR-0017).
**Fields:** `school_id`, `academic_year_id`, `assessment_period_id`, `subject_id`, `grade`,
`class_id?`, `section_id?` (null = grade+subject default coverage),
`chapters:[{chapter_id, status: NOT_STARTED|IN_PROGRESS|COMPLETED|EXCLUDED, completed_on?,
notes?, topics?:[{topic_id, status}]}]` (embedded, bounded),
`status: DRAFT|REVIEWED|LOCKED`, `locked_at?`, `locked_by?`, `updated_by`, timestamps.
**Indexes:** unique `(school_id, academic_year_id, assessment_period_id, subject_id,
class_id?, section_id?)`; `(school_id, assessment_period_id)`.
**Lifecycle:** DRAFT → REVIEWED → LOCKED. Coverage **inheritance**: class/section-specific
coverage wins; otherwise grade+subject default applies. Once LOCKED and referenced by an
exam scope, changes are audited and require a new exam version (syllabus lock).
**Modeling note:** embedded `chapters[]` per ADR-0017 (small, always read together) — not a
per-chapter collection.
**Resolution note (ADR-0017):** class/section-specific coverage is merged with the
grade+subject default **per chapter** (override status wins where both exist; default
status applies where the override omits the chapter). A partial override never implies
that unlisted chapters do not exist; scope resolution records per-chapter provenance
(`default` | `class_override`).

### `classes`
**Purpose:** Grade classes in a school-year.
**Fields:** `school_id`, `academic_year_id`, `grade:int`, `name`, `class_teacher_id:
null|ref.teachers`, timestamps.
**Indexes:** unique `(school_id, academic_year_id, grade, name)`.

### `sections`
**Purpose:** Sections within classes.
**Fields:** `class_id:ref.classes`, `section_code`, `teacher_ids[]`, `room_capacity`,
timestamps.
**Indexes:** unique `(class_id, section_code)`.

### `enrollments`
**Purpose:** Student↔class/section membership per year.
**Fields:** `school_id`, `academic_year_id`, `student_id:ref.students`, `class_id`,
`section_id`, `roll_no`, `status: ACTIVE|WITHDRAWN`, timestamps.
**Indexes:** unique `(school_id, academic_year_id, student_id)`; `(school_id,
academic_year_id, class_id, section_id)`; roll lookup `(class_id, roll_no)`.

### `subjects`
**Purpose:** Subjects; grade-bound; language metadata.
**Fields:** `school_id`, `academic_year_id`, `name`, `code`, `grade_levels:int[]`,
`languages:["EN","UR"]`, `is_language_subject`, `default_marks_config`, timestamps.
**Indexes:** unique `(school_id, academic_year_id, code)`.

### `chapters`
**Purpose:** Chapters under subject.
**Fields:** `school_id`, `subject_id`, `code`, `title`, `title_ur?`, `order:int`,
`learning_objectives:string[]`, `is_active`, timestamps.
**Indexes:** `(school_id, subject_id, order)`.

### `topics`
**Purpose:** Topics/subsections under chapter.
**Fields:** `school_id`, `chapter_id`, `title`, `title_ur?`, `order`,
`sections:[{title,order}]` (embedded), `learning_objectives[]`, timestamps.
**Indexes:** `(school_id, chapter_id, order)`.

---

## Learning materials

### `learning_materials`
**Purpose:** Uploaded materials + pipeline state + authority class.
**Fields:** `school_id`, `academic_year_id`, `title`, `material_type`
(`TEXTBOOK|NOTES|WORKSHEET|EXAM_REVIEW|OTHER`), `src_authority`
(`PRIMARY|SECONDARY|REFERENCE|PRACTICE`), `format` (MIME), `file_ref:{storage_key,
bucket, size_bytes, sha256, page_count}`, `uploaded_by`, `status`
(`UPLOADED|PROCESSING|ANALYZED|NEEDS_REVIEW|APPROVED|ARCHIVED|FAILED`),
`analysis?:{subject_id, grade, chapter, section, topic, confidence}`, `review:
{reviewed_by, reviewed_at, corrections}`, `languages[]`, `version:int`,
`error?:{code,message}`, timestamps.
**Indexes:** `(school_id,status)`; `(school_id, mapped subject after analysis)`;
`(school_id, src_authority)`; `file_ref.sha256` (upload dedupe); `uploaded_by`.
**Lifecycle:** [MATERIAL_INGESTION.md](MATERIAL_INGESTION.md). Only `APPROVED` is
RAG/exam-eligible.

### `material_pages`
**Purpose:** Per-page processing metadata.
**Fields:** `material_id`, `page_no`, `dpi`, `zones[]`, `ocr_needed`, `storage_ref`,
timestamps.
**Indexes:** unique `(material_id,page_no)`.

### `rag_chunks`
**Purpose:** Chunk registry — canonical text/metadata; the **rebuild source for Qdrant**.
**Fields:** `public_id` (ULID, unique), `school_id`, `academic_year_id`, `material_id`,
`subject_id`, `chapter_id`, `section_id`, `topic_id?`, `grade`, `language`,
`content_type` (`CONCEPT|DEFINITION|EXAMPLE|EXERCISE|FORMULA|NOTES|OTHER`),
`hierarchy_path[]`, `page_no`, `source_type`, `learning_objective`, `difficulty`,
`text`, `text_original_lang`, `metadata`, `embedding_status`
(`PENDING|INDEXED|FAILED|SUPERSEDED`), `embedding?:{model_id, model_version, collection,
point_id, last_indexed_at}`, `is_active`, timestamps.
**Indexes:** `(school_id,material_id)`; `(school_id,chapter_id)`;
`(school_id,embedding_status)`; unique `public_id`.
**Lifecycle:** inactive on re-chunk (new version); Qdrant points reference `public_id`.

### `material_processing_jobs`
**Purpose:** Per-stage pipeline status for materials.
**Fields:** `material_id`, `stage` (`VALIDATION|EXTRACT|OCR|ANALYZE|CHUNK|EMBED|INDEX`),
`status`, `attempts`, `last_error`, `started_at`, `finished_at`, timestamps.
**Indexes:** unique `(material_id, stage)`.

---

## RAG / model registries

### `embedding_model_registry`
**Purpose:** Registered embedding models — registry-driven; **bge-m3 is the MVP default,
not a hard-code** (ADR-0008).
**Fields:** `model_id` (e.g. `bge-m3`), `provider` (`local|ollama|cloud...`), `endpoint?`,
`vector_size` (bge-m3 = 1024 dense), `supports:["dense","sparse","colbert"]`,
`languages[]`, `model_version`, `metadata`, `is_default:bool`, `status:
ACTIVE|DEPRECATED`, timestamps.
**Indexes:** unique `model_id`; `(status,is_default)`.
**Lifecycle:** new version → new registry row; old rows DEPRECATED; re-index flow in
[QDRANT.md](QDRANT.md).

### `ai_model_registry`
**Purpose:** LLM/vision/OCR-capable model registry for the AI gateway.
**Fields:** `model_id`, `provider` (`openai|gemini|anthropic|glm|ollama|vllm|custom`),
`capabilities[]` (`text|structured|vision|embedding|rerank|translation|reasoning`),
`cost_profile:{input_per_1m, output_per_1m, currency}`, `context_window`,
`languages[]`, `max_output_tokens`, `supports_structured_output`, `status`, timestamps.
**Indexes:** unique `model_id`; `capabilities`.
**Lifecycle:** admin-managed; routing picks from ACTIVE (see
[MODEL_ROUTING.md](MODEL_ROUTING.md)).

### `model_runs`
**Purpose:** Per-invocation AI call record (evidence + audit + cost).
**Fields:** `public_id` (ULID), `instrument_ref:{type, id}` (question_id, grading_id, …),
`operation` (`question_generate|grade|ocr|translate|analyze|rerank|embed|key_generate|...`),
`task_id:ref.jobs?`, `provider`, `model_id`, `model_version`, `prompt_version`,
`input_tokens`, `output_tokens`, `image_analysis?:{count,size}`, `latency_ms`,
`estimated_cost`, `user_id`, `school_id`, `status` (`SUCCEEDED|FAILED|RETRIED`),
`response_ref?` (storage ref to raw response), `created_at`.
**Indexes:** `(school_id, created_at)`; `(instrument_ref.type, instrument_ref.id)`;
`(operation, created_at)`; `(model_id, created_at)`.
**Retention:** raw rows 1 year, monthly aggregates kept
(see [COST_MANAGEMENT.md](COST_MANAGEMENT.md)).

### `ai_usage`
**Purpose:** Aggregated AI cost ledger rows (rollups from `model_runs`).
**Fields:** `school_id`, `period` (`YYYY-MM`), `operation`, `provider`, `model_id`,
`input_tokens`, `output_tokens`, `image_count`, `estimated_cost`, `call_count`,
`p95_latency_ms`, `updated_at`.
**Indexes:** `(school_id, period, operation)`; `(period, provider)`.
**Retention:** monthly aggregates kept indefinitely; raw rows in `model_runs` archived
after 1 year (see [COST_MANAGEMENT.md](COST_MANAGEMENT.md)).

---

## Questions & rubrics

### `questions`
**Purpose:** Question bank entries (versioned, tenant-scoped).
**Fields:** `public_id` (ULID), `school_id`, `academic_year_id`, `subject_id`,
`chapter_id`, `section_id`, `topic_id?`, `question_type`
(`MCQ|MSQ|TRUE_FALSE|FILL_BLANK|SHORT|LONG|ESSAY|NUMERICAL|MATCHING|ORDERING|DIAGRAM|
EQUATION|TABLE|CUSTOM`), `text`, `language`, `difficulty: EASY|MEDIUM|HARD`,
`cognitive_level: REMEMBER|UNDERSTAND|APPLY|ANALYZE|EVALUATE|CREATE`, `marks:int`,
`options?:[{key,text,is_correct}]` (embedded; correct value RBAC-gated in API),
`correct_answer?` (per type; encrypted at rest option), `explanation`, `rubric: {
criteria:[{id,name,max_marks,descriptors[]}], total_marks}`, `source_refs[]`
(material/chapter/section/page), `creation_method` (`AI|MANUAL|IMPORT`), `evidence?`
(AI evidence chain — ADR-0014), `approval_status`
(`CANDIDATE|NEEDS_REVIEW|APPROVED|REJECTED|DEPRECATED`), `approved_by`, `approved_at`,
`origin_question_id?` (derived-from), `rev:int`, `usage_stats:{times_used,
correct_rate, discrimination?}`, timestamps.
**Indexes:** `(school_id, subject_id, chapter_id)`; `(school_id, question_type)`;
`(school_id, approval_status)`; `(school_id, difficulty)`; unique `public_id`;
text index on `text` for lexical dedupe; `sha256_canonical` unique-ish for exact dedupe.
**Lifecycle:** CANDIDATE → NEEDS_REVIEW → APPROVED / REJECTED; DEPRECATED on supersede.

### `question_versions`
**Purpose:** Immutable history of question edits.
**Fields:** `question_id` (`public_id`), `rev:int`, `snapshot` (full doc), `change_reason`,
`changed_by`, `changed_at`.
**Indexes:** `(question_id, rev)`.

### `rubrics`
**Purpose:** Standalone reusable rubrics (and exam-level marking schemes).
**Fields:** `school_id`, `name`, `subject_id?`, `criteria:[{id,name,max_marks,
descriptors[], scale}]`, `version:int`, `created_by`, timestamps.
**Indexes:** `(school_id, subject_id)`.

### `question_usage_stats`
**Purpose:** Analytics rollups for bank health.
**Fields:** `question_id` (`public_id`), `times_used`, `times_correct`, `p_value`,
`discrimination_index?`, `avg_time_sec?`, `updated_at`.
**Indexes:** `(question_id)`, `(p_value)`.

---

## Exams & integrity

### `exam_blueprints`
**Purpose:** Structural specification for exam generation (ADR-0017 extension).
**Fields:** `school_id`, `academic_year_id`, `assessment_period_id`, `examination_set_id?`,
`name`, `grade`, `subject_id`, `material_ids[]` (explicit allowlist — only these materials
are retrievable), `scope: {mode: NEW_ONLY|CUMULATIVE|SELECTED_CHAPTERS|FULL_SYLLABUS|CUSTOM,
base_period_id?, chapter_ids[], section_ids[]}`, `coverage_constraint: approved_only`,
`chapter_weighting:{chapter_id:pct}`, `section_weighting:{section_id:pct}`,
`weighting_tolerance_pct` (default 10, absolute percentage points), `learning_objectives[]`,
`total_marks`, `duration_min`,
`question_specs:[{question_type,count,marks_each,difficulty_distribution, blooms[]}]`,
`choice_groups?:[{id, label, question_count, attempt_count, marks_each}]` (internal
choice "attempt any N of M" — EXAM_ENGINE §3),
`language`, `instructions`, `scope_resolved_from` (`coverage_locked_snapshot_ref`),
`status: DRAFT|ACTIVE|ARCHIVED`, `creator`, `rev:int`, timestamps.
**Indexes:** `(school_id, subject_id)`; `(school_id, assessment_period_id)`; unique
`public_id`.
**Lifecycle:** `DRAFT → ACTIVE → ARCHIVED`. DRAFT: editable; scope can be changed and
(re-)resolved via `resolve-scope` from approved/locked coverage; may back a draft exam.
ACTIVE: usable blueprint revision — must not be silently mutated in ways that alter the
scope, question specifications, weighting, or other generation semantics of an existing
exam that has progressed beyond DRAFT (especially at/after READY); such changes require
a new blueprint revision (`rev++`, prior revision retained for audit). ARCHIVED:
retained for historical/audit; not used for new exams. Scope is revalidated at
generation (fail closed); **every generation run records the blueprint `rev` used** (in
evidence). Scope-mode resolution algorithms and the generation-eligibility rule:
EXAM_ENGINE §2b (`base_period_id` = earliest assessment period to include in CUMULATIVE
resolution, same academic year only).

### `examination_sets`
**Purpose:** Groups the exams of one assessment period (ADR-0017). e.g., Grade 8 Mid-Term
Set → Mathematics, Physics, English, Urdu, Chemistry exams.
**Fields:** `public_id`, `school_id`, `academic_year_id`, `assessment_period_id`, `name`,
`grade`, `class_ids[]`, `section_ids[]`, `subject_ids[]`, `status:
DRAFT|SCHEDULED|PUBLISHED|COMPLETED|ARCHIVED`, `schedule:{start_datetime?, end_datetime?}`
(advisory in MVP; hard enforcement is a school-config option), `published_at`,
`published_by`, `rev:int`, timestamps.
**Indexes:** unique `(school_id, academic_year_id, assessment_period_id, grade, name)`;
`(school_id, status)`; `(school_id, assessment_period_id)`.
**Lifecycle:** DRAFT → SCHEDULED → PUBLISHED → COMPLETED → ARCHIVED. Publishing the set
does not auto-publish member exams (each exam keeps its own publish/lock lifecycle).

### `exams`
**Purpose:** Exam workspace/draft; versions are immutable. (ADR-0017 extension)
**Fields:** `public_id`, `school_id`, `academic_year_id`, `assessment_period_id`,
`examination_set_id?`, `blueprint_id?` (exam ↔ blueprint link), `exam_type`
(`QUIZ|CLASS_TEST|MONTHLY_TEST|MIDTERM|FINAL|MOCK|ASSIGNMENT|HOMEWORK|PRACTICAL|VIVA`),
`name`, `subject_id`, `class_ids[]`, `duration_min`, `total_marks`, `status`
(`DRAFT|READY|PUBLISHED|ARCHIVED`), `current_version:int`, `question_ids[]`
(`public_id` refs in order), `answer_key_status` (`DRAFT|APPROVED`),
`syllabus_snapshot` (frozen scope/coverage hash at READY; syllabus lock), `published_at`,
`published_by`, `rev:int`, timestamps.
**Indexes:** `(school_id, status)`; `(school_id, class_ids)`; `(school_id,
assessment_period_id)`; `(examination_set_id)`; unique `public_id`.
**Lifecycle:** DRAFT → READY (syllabus frozen) → PUBLISHED (locked) → ARCHIVED. Scope/
coverage changes after READY require a new version.

### `exam_versions`
**Purpose:** Immutable published snapshot.
**Fields:** `exam_id`, `version:int`, `snapshot` (full serialized exam incl. questions),
`scope_snapshot` (resolved scope + coverage hash — enables syllabus lock rollback/audit),
`checksum_sha256`, `published_by`, `published_at`, `change_notes`, timestamps.
**Indexes:** unique `(exam_id, version)`.

### `exam_paper_instances`
**Purpose:** Per-printed-paper machine-readable identity (integrity).
**Fields:** `public_id` (`paper_id`, ULID), `exam_id`, `version`, `school_id`,
`student_id?`, `order_seed`, `question_order[]` (shuffled, referential),
`option_order:{question_id: [order]}` (embedded), `qr_payload`, `status`
(`GENERATED|PRINTED|RETURNED|PROCESSING|GRADED|VERIFIED|VOID`), `returns:
[{scan_ref, arrived_at}]`, `checksum`, timestamps.
**Indexes:** `(exam_id, status)`; unique `public_id`; `(school_id, qr_payload)`.
**Security:** ordering is school-private; key-access audited (below).

### `exam_answer_keys`
**Purpose:** Secure, RBAC-gated answer keys per version.
**Fields:** `exam_id`, `version`, `key` (encrypted at rest), `approval:{status,by,at}`,
`access_policy` (`role_required`), timestamps.
**Indexes:** unique `(exam_id, version)`.
**Access:** only via `exams:key:read`; every access → `exam_access_audit`.

### `exam_access_audit`
**Purpose:** Immutable log of key/paper access.
**Fields:** `exam_id`, `paper_id?`, `action` (`KEY_VIEW|KEY_DOWNLOAD|PAPER_PRINT|
PAPER_REGENERATE|KEY_CORRECT`), `user_id`, `school_id`, `ip`, `user_agent`, `at`.
**Indexes:** `(exam_id, at)`; `(user_id, at)`.

---

## Attempts, OCR & grading

### `student_attempts`
**Purpose:** One student's submitted exam.
**Fields:** `public_id`, `school_id`, `exam_id`, `exam_version`, `assessment_period_id`,
`paper_id: null|ref.exam_paper_instances`, `student_id`, `class_id`, `section_id`, `status`
(`UPLOADED|PROCESSING|GRADING|VERIFICATION|COMPLETE|FAILED`), `scan_refs:[{storage_key,
page_no, arrived_at}]`, `language_detected?`, `total_awarded?`, `finalized_at?`,
`verified_by?`, timestamps.
**Indexes:** `(school_id, exam_id, status)`; `(school_id, student_id)`; `(school_id,
assessment_period_id)`; unique `public_id`.

### `student_answers`
**Purpose:** Reconstructed per-question answer for an attempt (immutable originals kept).
**Fields:** `attempt_id` (`public_id`), `question_id`, `status`
(`RAW|NORMALIZED|TRANSLATED|GRADED|VERIFIED`), `original_ref?` (storage ref to crop),
`answer_type`, `raw_text?`, `normalized_text`, `detected_language`, `translated_text?`
(stored separately, original preserved), `translation_confidence?`, `recognition
confidence?`, `question_ref_mapping_conf`, `evidence_refs[]`, timestamps.
**Indexes:** `(attempt_id, question_id)` unique; `(question_id, status)`.

### `ocr_results`
**Purpose:** Per-region OCR/handwriting output + confidence + provenance.
**Fields:** `attempt_id`, `question_id`, `region_ref` (storage key of crop), `engine:
{provider, model, version}`, `mode` (`PRINTED|HANDWRITTEN|MIXED`), `raw_output`,
`normalized_output`, `detected_language`, `confidence`, `consensus?:{models[], votes[],
final, method}`, `status` (`CANDIDATE|ACCEPTED|CORRECTED|REJECTED`), `corrected_by?`,
`corrected_at?`, timestamps.
**Indexes:** `(attempt_id, question_id)`; `(engine.provider, status)`.

### `translations`
**Purpose:** Optional answer translations (originals always preserved).
**Fields:** `attempt_id`, `question_id`, `source_lang`, `target_lang`, `source_text_ref`
(crop/ocr ref), `translated_text`, `confidence`, `provider`, `provider_model`, `status`
(`CANDIDATE|ACCEPTED|REJECTED`), timestamps.
**Indexes:** `(attempt_id, question_id)`.

### `grading_results`
**Purpose:** Grading per answer — candidate + final + evidence + confidence.
**Fields:** `public_id`, `attempt_id`, `question_id`, `student_id`, `exam_id`, `mode`
(`OBJ_DETERMINISTIC|AI_RUBRIC|MANUAL`), `candidate:{marks, criteria_marks[], explanation,
confidence, provider, model, prompt_version, raw_ref}` (AI candidate — never final),
`final_marks?`, `final_by?`, `final_at?`, `status`
(`PENDING|CANDIDATE|NEEDS_REVIEW|APPROVED|REJECTED`), `verification_reason`, `evidence`
(embedded chain — see ADR-0014), `retry_count`, timestamps.
**Indexes:** `(attempt_id, question_id)` unique; `(status, school_id)` — verification
queue; `(question_id)` — question stats; `(student_id)`.

### `teacher_verification_items`
**Purpose:** Work queue for teacher review.
**Fields:** `grading_id` (`public_id`), `attempt_id`, `paper_ref`, `crop_ref`, `ocr_ref`,
`translation_ref`, `question_ref`, `key_ref`, `rubric_ref`, `context_refs[]`, `reason`
(`LOW_CONFIDENCE|FLAGGED|TAMPER|AMBIGUOUS`), `assigned_to?`, `status`
(`QUEUED|IN_REVIEW|APPROVED|MODIFIED|REJECTED`), `decision?, decision_note`,
`handled_by`, `handled_at`, timestamps.
**Indexes:** `(status, school_id)`; `(grading_id)` unique.

### `evidence_snapshots`
**Purpose:** Immutable, complete AI decision packages (audit/explainability).
**Fields:** `public_id`, `instrument_ref:{type,id}`, `inputs_composed` (hash + refs),
`retrieved_chunks[]`, `prompt_version`, `model_runs[]`, `raw_outputs_ref`, `created_at`.
**Indexes:** `(instrument_ref.type, instrument_ref.id)` unique; `created_at`.
**Retention:** platform lifetime (mandatory auditability).

---

## Results & analytics

### `grading_scales`
**Purpose:** School-configurable grading scales mapping percentage → grade (no
jurisdiction-specific system hard-coded).
**Fields:** `school_id`, `name`, `applicability:{grade_levels?:int[], subject_ids?:[],
academic_year_id?}` (absent = school default), `bands:[{grade, min_pct, max_pct?,
pass:bool}]` (bands cover the complete 0–100 percentage range; omitted `max_pct` = open
upper bound), `is_default:bool`, `status: ACTIVE|ARCHIVED`, timestamps.
**Indexes:** unique `(school_id, name)`; `(school_id, is_default)`.
**Resolution precedence:** subject-specific > grade-level-specific > school default.
**Grade computation:** `percentage → applicable grading scale → matching band
(`min_pct ≤ percentage ≤ max_pct`) → grade / pass-fail (band's `pass` flag)`.
If no applicable grading scale exists: `student_results.grade = null` (never guessed) and
a warning surfaces in results/report-card generation. `student_results.grade` and report
cards are computed from this scale.

### `student_results`
**Purpose:** Approved per-exam student outcome.
**Fields:** `school_id`, `attempt_id`, `exam_id`, `academic_year_id`, `assessment_period_id`,
`examination_set_id?`, `student_id`, `class_id`, `total_marks`, `obtained_marks`,
`percentage`, `grade?` (computed from the applicable `grading_scales` entry),
`question_results[]` (compact), `calculated_at`, `finalized`,
timestamps.
**Indexes:** `(school_id, exam_id, class_id)`; `(school_id, student_id)`; `(school_id,
academic_year_id, assessment_period_id)`; unique `attempt_id`.

### `class_results`
**Purpose:** Class-level aggregate per exam.
**Fields:** `school_id`, `exam_id`, `academic_year_id`, `assessment_period_id`,
`examination_set_id?`, `class_id`, `section_id?`, `average`, `median`, `max`, `min`,
`pass_rate`, `count`, `stat_kind` (`CLASS|SECTION`), `computed_at`.
**Indexes:** `(school_id, exam_id, class_id)`; `(school_id, assessment_period_id)`.

### `subject_results`
**Purpose:** Subject performance.
**Fields:** `school_id`, `academic_year_id`, `assessment_period_id`, `examination_set_id?`,
`subject_id`, `student_id?`, `class_id?`, `average`, `count`, `stat_kind`
(`STUDENT|CLASS|SUBJECT`), `computed_at`.
**Indexes:** `(school_id, subject_id)`; `(school_id, assessment_period_id)`.

### `chapter_performance`
**Purpose:** Chapter/topic mastery analytics.
**Fields:** `school_id`, `exam_id?`, `class_id?`, `subject_id`, `chapter_id`, `topic_id?`,
`assessment_period_id`, `academic_year_id`, `avg_correct_rate`, `student_id?`, `stat_kind`,
`computed_at`.
**Indexes:** `(school_id, subject_id, chapter_id)`; `(school_id, assessment_period_id)`.

### `analytics_cubes`
**Purpose:** Precomputed analytical dimensions (teacher/school/trends).
**Fields:** `school_id`, `dimension` (`STUDENT|CLASS|QUESTION|TEACHER|SCHOOL|PERIOD`),
`period` (`WEEK|MONTH|TERM|YEAR`), `assessment_period_id?`, `key` (student_id/class_id/...),
`payload` (aggregates, histograms), `computed_at`, `stale:bool`.
**Indexes:** `(school_id, dimension, period, key)`; `(school_id, stale)`.

---

## Jobs, audit, eval & meta

### `jobs`
**Purpose:** Generic job registry for all background work.
**Fields:** `job_id` (implementation key), `job_type`, `queue`, `payload_ref`,
`status: QUEUED|RUNNING|SUCCEEDED|FAILED|CANCELED`, `attempts`, `max_attempts`,
`last_error`, `trace_id`, `retryable:bool`, `progress:0-1`, `started_at`, `finished_at`,
`heartbeat_at`, timestamps.
**Indexes:** `(status, queue)`; `(job_type, created_at)`; unique `job_id`.

### `audit_logs`
**Purpose:** Append-only audit trail (all domains).
**Fields:** `actor_user_id`, `actor_role`, `school_id`, `action`, `resource_type`,
`resource_id`, `before?`, `after?`, `ip`, `user_agent`, `trace_id`, `hash_prev`
(tamper-evident hash chain), `at`.
**Indexes:** `(resource_type, resource_id)`; `(actor_user_id, at)`; `(school_id, at)`.
**Retention:** platform lifetime.

### `notifications`
**Purpose:** In-app/user notifications (review-queue items, job failures).
**Fields:** `user_id`, `school_id`, `type`, `payload`, `read:bool`, `created_at`.
**Indexes:** `(user_id, read)`; TTL optional.

### `eval_datasets`
**Purpose:** Golden dataset registry (AI Evaluation Lab).
**Fields:** `code` (unique, e.g. `rag_grade9_subject`), `capability`
(`RAG|OCR|HW_EN|HW_UR|TRANSLATION|QG|GRADING`), `version`, `storage_ref`,
`metrics_config`, `splits:{train,dev,test}`, `owner`, timestamps.
**Indexes:** unique `code`.

### `eval_runs`
**Purpose:** Regression/eval run results.
**Fields:** `dataset_code`, `capability`, `revision` (`backend`/`app`/`prompt` hashes),
`configuration`, `metrics` (e.g. hit@5, CER, MAE), `pass:bool`, `thresholds`,
`artifacts_ref`, `trigger`, `ran_by`, timestamps.
**Indexes:** `(capability, dataset_code, ran_at)`; `(pass)`.

### `schema_migrations`
**Purpose:** Migration bookkeeping.
**Fields:** `name` unique, `applied_at`, `checksum`.
**Indexes:** unique `name`.