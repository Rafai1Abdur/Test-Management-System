# Material Ingestion — Pipeline, Lifecycle, Formats, Authority

## 1. Pipeline overview

```
Upload → validation → security check → format detection → content extraction → OCR/vision
→ normalization → language detection → structure detection → subject/grade/chapter/section/
topic detection → semantic/hierarchical chunking → chunk persistence (MongoDB) → embeddings
→ Qdrant indexing → quality validation → review → approval → ARCHIVED (later)
```

Each stage is a worker step in `material_processing_jobs` with status/progress/retry
([QUEUES_WORKERS.md](QUEUES_WORKERS.md)).

## 2. Upload & validation

- Authorized roles: Teacher, Principal, Admin, Super Admin (school-scoped).
- API: `POST /api/v1/materials` (multipart, metadata). Accepts the format list below.
- Validation rules: MIME + magic-byte check; size limit (default 200 MB); extension
  allowlist; malware scan (ClamAV in MVP; async, must pass before extraction);
  PDF/DOCX macro & script stripping where applicable ([SECURITY.md](SECURITY.md)).

## 3. Format adapters (extensible)

| Format | Adapter strategy (MVP) |
|---|---|
| PDF | pypdf/pdfplumber text+geometry; OCR fallback for scanned pages (Tesseract/PaddleOCR) |
| DOCX / DOC | python-docx (styles/headings for TOC); DOC via LibreOffice conversion (headless) |
| PPTX / PPT | python-pptx (slide titles + notes); LibreOffice for PPT legacy |
| TXT / Markdown | direct parse; MD headings → structure |
| HTML | html2text/BeautifulSoup; heading hierarchy |
| EPUB | ebooklib → chapters → XHTML → text |
| JPG/JPEG/PNG/WEBP/TIFF | vision/OCR pipeline (single or multi-page TIFF) |

New formats = new adapter implementing `ContentExtractor` — no pipeline changes.

## 4. Processing stages

1. **Extraction**: adapter → pages/units with layout signals (headings, numbering).
2. **OCR/vision**: required when text layer absent (scan); per-page; region-aware; OCR
   confidence recorded (`ocr_results`-style within material processing).
3. **Normalization**: Unicode normalization (e.g., Urdu Nastalique variants), whitespace,
   page numbering alignment.
4. **Language detection**: fastText/Franc-style; per-document & per-unit
   (language per chunk).
5. **Structure detection**: TOC/heading detection → chapter/section/topic tree proposals.
6. **Curriculum mapping (AI candidate)**: subject/grade/chapter/section/topic prediction
   grounded in existing curriculum + materials; **candidate confidence**; teacher must
   confirm/correct (NEEDS_REVIEW).
7. **Chunking**: ([CHUNKING.md](CHUNKING.md)) → `rag_chunks`.
8. **Embedding + indexing**: embed (registered model) → Qdrant collection
   ([QDRANT.md](QDRANT.md)) → `embedding_status=INDEXED`.
9. **Quality validation**: coverage checks, garbage-chunk scan, duplicate-rate audit.
10. **Review & approval**: teacher sees analysis + chunk preview; approves →
    `APPROVED` (only then RAG-eligible), or requests corrections (→ `NEEDS_REVIEW`),
    or rejects (→ `FAILED`/`ARCHIVED`).

## 5. Lifecycle states

```
UPLOADED → PROCESSING → ANALYZED → NEEDS_REVIEW → APPROVED → ARCHIVED
               │   (per-stage)         │  ▲            │
               ▼                        ▼  │            ▼
             FAILED ──────────────────►(retryable)   (rejected → ARCHIVED)
```
- `ANALYZED`: analysis + chunks ready, awaiting teacher.
- `NEEDS_REVIEW`: auto-analysis low-confidence or teacher corrections requested.
- `FAILED`: records `error.code/message`; retryable unless non-retryable cause.
- Transitions are validated; audit events at every transition.

## 6. Source authority (exam-scope control)

| Class | Meaning | Exam default |
|---|---|---|
| `PRIMARY` | Official textbook/curriculum source | included |
| `SECONDARY` | Teacher notes, sanctioned supplements | included |
| `REFERENCE` | Background/enrichment | excluded unless selected |
| `PRACTICE` | Worksheets/past papers | excluded from generation; usable in practice mode |

Stored on `learning_materials.src_authority`; enforced by `RetrievalScope` in generation.

## 7. Versioning & re-approval

- New upload of same title → new version; existing evidence/refs keep old material id.
- Edits to analysis metadata do **not** re-embed unless chunks change; chunk-affecting edits
  trigger re-chunk + re-embed + alias-safe index update.
- Approval is per version.

## 8. Failure & retry table

| Failure | Handling |
|---|---|
| Malware detected | Reject upload; quarantine; audit + notify admin |
| Unreadable file / zero text | FAILED(NON_RETRYABLE) with hint |
| OCR provider down | FAILED(RETRYABLE); alternate provider route |
| Embedding/indexing fail | Retry; if persistent → material stays ANALYZED, deferred embed job |
| Analysis low confidence | → NEEDS_REVIEW (never auto-approved) |