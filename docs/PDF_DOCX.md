# PDF & DOCX Generation

## 1. Principle: canonical model → template → output

Both formats are derived from the **canonical exam domain model** (ADR-0011,
[EXAM_ENGINE.md](EXAM_ENGINE.md)) — never authored independently.

```
Exam Domain Model → Normalized Render Tree → Template Renderer
                                            ├── DOCX (python-docx / docxtpl)
                                            └── PDF  (HTML/Jinja2 → WeasyPrint or
                                                      wkhtmltopdf/Chromium)
```

## 2. Render tree

A format-neutral tree: `ExamHeader, Branding, Sections, QuestionBlocks, AnswerSpaces,
PageNumbers, Footer, QR/Barcodes, Bilingual Blocks` — validated before render (marks sums,
duration, per-paper orderings, question count).

## 3. Features supported

- School branding: logo (object-storage ref), name, watermark/background
- Exam title, subject, grade, assessment period, duration, marks, section headers
- **Report cards** (Phase 9): per (student, assessment period, examination set); school-branded
  configurable templates with a platform default fallback; grades rendered from
  `student_results.grade` computed via the school's `grading_scales` (MONGODB_SCHEMA);
  same render tree ([ADR-0017](ADR/ADR-0017-assessment-periods-coverage-exam-sets.md))
- Student info fields (name/roll/class/paper_id)
- Instructions (per section), page numbers, headers/footers
- Question numbering (continuous or per-section); marks column; answer spaces sized by marks
- Bilingual layouts (EN + UR; RTL handling, Urdu fonts embedded)
- Machine-readable elements: paper_id, QR/barcode, question identifier margins, checksums
  ([EXAM_INTEGRITY.md](EXAM_INTEGRITY.md))
- Answer **sheets**: pre-printed answer regions aligned to processing templates; QR links
  paper to regions

## 4. Format specifics

### PDF
- HTML/CSS templates + WeasyPrint (local, deterministic, decent RTL via proper CSS
  `unicode-bidi`/fonts). Fallback drivers: Chromium print.
- Embedded fonts for Urdu (e.g., Noto Naskh Arabic); barcode/QR rendered clientless
  (python-qrcode / reportlab or in HTML as SVG).
- Output to object storage `exams/`; signed download URLs.

### DOCX
- python-docx/docxtpl with styles (school-branded docx templates .docx as Jinja templates).
- RTL runs for Arabic-script segments; fonts embedded on open.
- Same render tree so DOCX matches PDF structure; docx→pdf equivalence tested in CI.

## 5. Answer-key sheet & reports

- Key sheets (RFC: teacher's copy) alternate layout but same content; RBAC-gated access.
- Results reports (student/class) reuse the same template engine
  (`reports/` role in STORAGE.md). **Report cards** per (student, assessment period, exam set)
  use the same pipeline — school-branded template config with platform default fallback
  ([ADR-0017](ADR/ADR-0017-assessment-periods-coverage-exam-sets.md)).

## 6. Rendering as background jobs

- `worker-export` renders PDF/DOCX; idempotent per (exam_id, version, paper_id, format);
  progress + failure states recorded in `jobs`; retries on transient errors.
- Render heavy pages: cache rendered artifacts by content hash to avoid re-render.

## 7. Quality & testing

- Golden layout tests: structure assertions, checksum regex, marks totals, page counts,
  RTL render smoke tests (Urdu glyph shapes), QR decodability on printed-layout fixtures
  ([TESTING.md](TESTING.md)).