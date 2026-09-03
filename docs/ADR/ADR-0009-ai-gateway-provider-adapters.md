# ADR-0009 — AI Gateway with Provider Adapters (No Provider Lock-in)

- **Date:** 2026-09-02
- **Status:** Approved

## Context

Product requires support for free/paid/local/cloud/self-hosted models across multiple
providers (GLM, OpenAI, Gemini, Anthropic, Ollama, vLLM, others) and must not hard-code
provider logic. Capabilities (text, structured, vision, ocr, embeddings, rerank,
translation, reasoning) should be orthogonal to providers.

## Decision

1. Domain code depends only on **capability interfaces** (AI_PROVIDER_GATEWAY.md §2).
2. A **gateway** maps capability calls → provider adapters selected by the router
   (MODEL_ROUTING.md), with retries/failover across eligible providers and full run
   recording (`model_runs`, cost, audit).
3. Model registry rows describe capabilities/languages/cost/context/vision/structured
   support; routing honors data-policy hard filters (e.g., student answers → local only).
4. Provider adapters are thin, versioned, testable with recorded fixtures.

## Consequences

- **Positive:** flexibility, resilience, cost control, offline-capable option; eval can
  compare models.
- **Negative:** adapter surface must be maintained; behavior differences must be normalized
  (structured output, token units).

## Invalidation triggers

A new provider that cannot fit the capability interface — extend interface; otherwise the
gateway pattern is stable.