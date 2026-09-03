# Handwriting Recognition — English & Urdu, Multi-Model, Verified

## 1. Position & honesty

We do **not** claim perfect recognition of arbitrary handwriting. The architectural
requirement, from day one, is:

> Multi-model handwriting recognition **with confidence scoring and human verification**,
> prioritizing **English and Urdu**, extensible to further languages.

Urdu handwriting is explicitly **in scope architecturally from the start** (Phase 1 design;
Phase 2 delivery) — same pipeline, stricter gates, extra evaluation data.

## 2. Multi-model consensus pipeline

For each handwritten answer region:

```
crop (from paper pipeline) → pre-processing (normalize, line-segmentation)
  → [model A: Tesseract-esque/HTR local]   (e.g., PaddleOCR-UR, model per lang)
    [model B: HTR/VLM vision model]        (local or cloud; policy-gated)
    [model C: VLM with context prompt]     (question-aware)
  → decode → normalize (Unicode, RTL) → language detection
  → consensus voting (edit-distance weighted; majority/weighted by model trust)
  → final text = consensus winner (or top-N hypotheses)
  → confidence = agreement + per-model confidences
```

- Output: `consensus:{models[], votes[], final, method, agreements}` on `ocr_results`
  (mode=`HANDWRITTEN`).
- If models disagree strongly (agreement < threshold): keep **top hypotheses** for the
  teacher; do not pick arbitrarily.

## 3. Confidence & thresholds

| Lane | Default auto-accept* | Below → mandatory review |
|---|---|---|
| English handwriting | 0.70 | flag |
| **Urdu handwriting** | 0.60 (strictest) | flag with all hypotheses |
| *auto-accept = only becomes *teacher-acceptable*; teacher bulk-approval still required by policy (AI never final) | | |

Thresholds are configurable per school/task. **No handwriting output is ever fed to
grading without a confidence + status flag**; low-confidence regions show the crop in the
teacher verification UI with ALL hypotheses side-by-side.

## 4. English handwriting (MVP lane, Phase 1 design / Phase 6-7)

- Engines: local HTR (PaddleOCR-en), VLM (e.g., Qwen-VL) consensus; Tesseract as weak
  fallback/word-clues.
- Normalization: standard spelling normalization for grading (numeric vs words).
- EVAL: golden EN handwriting dataset; CER/WER gates.

## 5. Urdu handwriting (Phase 2 lane — same architecture)

- Engines: PaddleOCR-UR / Urdu-capable HTR models; VLM with Urdu prompt pack; consensus.
- Special handling: Nastalique joining (context-sensitive), diacritics/izafat, RTL line
  segmentation, Urdu-digit ↔ Latin-digit normalization (store both).
- **No perfect recognition claim**; gates + verification mandatory; evaluation dataset is a
  Phase 2 deliverable ([ROADMAP.md](ROADMAP.md) Phase 7, [AI_EVALUATION.md](AI_EVALUATION.md)).

## 6. Storage of evidence (never delete originals)

Per region store:
- original crop (object storage — immutable)
- per-model raw output + confidence
- consensus result + agreement
- detected language; normalized text
- status + corrections history

All stored in `ocr_results`, referencing immutable crops. Teacher corrections create
`CORRECTED` records referencing the originals.

## 7. Extensibility

- New language = new `HandwritingProvider` adapter + engine pack + normalization rules +
  evaluation dataset + confidence table entry. Interface unchanged
  ([AI_PROVIDER_GATEWAY.md](AI_PROVIDER_GATEWAY.md)).

## 8. Integration guardrails

- Grading consumes the **normalized text + confidence + status**; low-confidence or
  `NEEDS_REVIEW` answers are not auto-graded (they enter the verification queue with the
  crop so the teacher grades from the original).
- Translation (if enabled) always runs on the accepted/verified text; original preserved
  ([LANGUAGE_TRANSLATION.md](LANGUAGE_TRANSLATION.md)).