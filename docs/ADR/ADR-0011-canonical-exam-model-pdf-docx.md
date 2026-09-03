# ADR-0011 — Canonical Exam Model → Template → PDF / DOCX

- **Date:** 2026-09-02
- **Status:** Approved

## Context

PDF and DOCX must never be generated from unrelated content; both derive from one exam
representation (product rule). Exam content includes branding, bilingual layouts, answer
spaces, QR/barcodes, paper instances.

## Decision

1. **Canonical exam domain model** (EXAM_ENGINE.md §3) is the single source of truth for
   rendering all formats.
2. A **normalized render tree** is derived once per (exam, version, paper, language) and
   rendered to PDF (Jinja2 HTML + WeasyPrint, fallback Chromium) and DOCX (python-docx /
   docxtpl), producing matching structure.
3. Rendering runs in `worker-export`; artifacts cached by content hash; outputs stored in
   object storage; signed download URLs.
4. Bilingual/Urdu rendering via RTL CSS + embedded fonts; QR/barcodes generated in-render.

## Consequences

- **Positive:** guaranteed format parity; single render pipeline to test; integrity features
  (paper instances, randomization) naturally apply to both formats.
- **Negative:** HTML/Word fidelity differences require golden layout tests (addressed in
  TESTING.md); WeasyPrint/Urdu font handling needs care.

## Invalidation triggers

A new output format (e.g., HTML live exam) — add renderer to the same render tree.