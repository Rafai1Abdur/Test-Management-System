# OCR — Printed Text, Machine-Readable Exams, Confidence

OCR serves two flows: (1) scanned learning-material pages without a text layer, and
(2) scanned/photo **student exam papers**. This document covers the OCR architecture;
handwriting specifically in [HANDWRITING.md](HANDWRITING.md).

## 1. OCR provider abstraction (ADR-0010)

`OCRProvider` capability interface. Adapters (MVP):

| Engine | Type | Strength | Use |
|---|---|---|---|
| Tesseract (`eng`, `urd` traineddata) | local/free | printed EN/UR, simple layout | default printed OCR |
| PaddleOCR | local/free | robust layout, printed + batch | scanned pages, tables |
| VLM adapter (Qwen-VL / gemini vision…) | local/cloud | complex layouts, contextual | fallback + ambiguous regions |
| Cloud vision OCR (Azure/Google/GCP) | cloud | high accuracy | optional paid tier (data-policy gated) |

Selection via model router (`ocr` capability). **No engine hard-coded in business logic.**

## 2. Image pre-processing pipeline (per page)

```
deskew/rotation → perspective correction (QR/corner anchors) → crop margins →
denoise/shadow-removal → binarize → DPI normalize (≥300 for OCR) →
region detection → per-region recognition
```

For machine-readable papers, QR/barcode decoding gives paper_id + question geometry
→ region templates (~exact) — dramatically improves throughput
([EXAM_INTEGRITY.md](EXAM_INTEGRITY.md)).

## 3. Region detection

- **Template-driven** (machine-readable papers): known question/answer regions from the
  paper instance.
- **Heuristic** (free-form): rule-of-lines, table zones, answer boxes; VLM fallback.
- Each region → `ocr_results` row with crop ref, engine, confidence, status.

## 4. Confidence & status

- Per-region + per-word confidence from engines; normalized to `0..1`.
- Region-level `confidence` = aggregate (min/mean weighted).
- Status: `CANDIDATE → ACCEPTED | CORRECTED | REJECTED` (teacher repair in review UI).
- Gate: OCR confidence below task threshold (`printed` default 0.80) →
  `NEEDS_REVIEW` for the question; teacher sees crop + raw OCR side-by-side.

## 5. Language handling

- Detect per region (language detector) → route to matching engine pack
  (`urd` vs `eng`). Store `detected_language` and `normalized_output` (Unicode
  normalization, RTL handling with proper reordering at output assembly).

## 6. Machine-readable exams synergy

- QR/barcode encodes `{exam_id, version, paper_id, page_no}` + checksum.
- Returned page decode → attempt auto-linking; per-question regions located by template
  → per-question crops stored as evidence.

## 7. Outputs & storage

- Original page images: object storage, immutable.
- Crops: object storage (`papers/` layout) with refs in `ocr_results`.
- Raw engine output (boxes, words, confidences) persisted (JSON/JSONL stored or inline
  compact), enabling re-processing without re-running.

## 8. Metrics

- CER/WER on golden datasets ([AI_EVALUATION.md](AI_EVALUATION.md)); time per page;
  region-level success rate; teacher-correction rate as live proxy
  ([OBSERVABILITY.md](OBSERVABILITY.md)).