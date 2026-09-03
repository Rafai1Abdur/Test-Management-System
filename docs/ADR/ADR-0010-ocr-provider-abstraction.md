# ADR-0010 — OCR / Handwriting Provider Abstraction

- **Date:** 2026-09-02
- **Status:** Approved

## Context

OCR must support printed English and Urdu, handwriting (EN/UR), and be extensible to more
languages — without hard-coding one OCR engine.

## Decision

1. Define `OCRProvider` and `HandwritingProvider` capability interfaces (gateway, ADR-0009)
   with adapters: Tesseract (`eng`/`urd`), PaddleOCR, and VLM-based adapters (local or
   cloud per data policy).
2. Handwriting runs a **multi-model consensus** pipeline with per-model confidence,
   agreement-based selection, and kept hypotheses (HANDWRITING.md §2).
3. Every region produces: crop ref, per-engine outputs, language, confidence, status
   (CANDIDATE/ACCEPTED/CORRECTED/REJECTED); originals immutable; teacher repair workflow.
4. New languages = new adapter/pack + normalization + eval dataset — no pipeline changes.

## Consequences

- **Positive:** no vendor lock-in; graceful degradation (local engines first); measurable
  quality per engine via eval.
- **Negative:** consensus logic + normalization adds complexity; engine-specific tuning
  needed (esp. Urdu Nastalique).

## Invalidation triggers

A clearly superior dedicated OCR service (still an adapter); no known issues for the
interface.