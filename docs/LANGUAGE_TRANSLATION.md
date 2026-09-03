# Language & Translation Pipeline

## 1. Principle

**The original student answer is always preserved.** Translation assists understanding and
grading; it never replaces the original.

## 2. Pipeline

```
answer region (crop) → [handwriting/text OCR] → recognize text
→ normalize (Unicode, RTL, Urdu diacritics, digit normalization preserving both scripts)
→ language detection (per region; fastText/Franc + domain heuristics)
→ store: original_ref (crop), recognized_text, detected_language, recognition_confidence
→ [optional] translation (teacher/school policy: on/off, source/target, provider)
→ store: translated_text, translation_confidence, target_language (separate field)
→ status: CANDIDATE → ACCEPTED (teacher may correct)
```

## 3. Data contract (always-kept fields)

| Field | Meaning | Never overwritten by |
|---|---|---|
| `original_ref` | immutable crop storage ref | anything |
| `recognized_text` | OCR/HTR output (normalized) | translation |
| `detected_language` | per region | — |
| `translated_text` | optional translation | original recognized text |
| `translation_confidence` | 0..1 | — |
| `status` | CANDIDATE/ACCEPTED/CORRECTED | — |

## 4. Translation specifics

- Source languages MVP: `UR → EN` (grade in English) and `EN → UR` (teacher review);
  generic pairs via gateway `translation` capability.
- Routing: local NLLB/OPUS translators (free, private) or cloud (policy-gated); LLM
  translation adapter with guardrail prompt (preserve math/technical terms).
- Confidence: model score; below threshold → translation tagged low-confidence and the
  teacher sees original + rough-translation cues, never a silent auto-use.
- Grading: uses verified original text (or teacher-approved translation); the AI grader
  records which (original/translation) was used in the evidence chain.

## 5. RTL & display

- Output layer (frontend) handles RTL via `dir=rtl` and proper bidi; backend stores logical
  order; PDF/DOCX generation renders Urdu via RTL-capable fonts
  ([PDF_DOCX.md](PDF_DOCX.md)).

## 6. Evaluation

- Translation eval on golden pairs (`AI_EVALUATION.md`): semantic similarity, terminology
  retention; automatic metrics (chrF/COMET-style) + human spot-checks.