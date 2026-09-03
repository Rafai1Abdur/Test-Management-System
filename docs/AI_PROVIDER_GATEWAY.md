# AI Provider Gateway — Adapter Architecture

## 1. Goal

**Zero provider lock-in.** Domain code never calls a provider SDK. It calls capability
interfaces; a gateway maps interfaces → providers (cloud, local, self-hosted, free/paid).

## 2. Interfaces (capabilities)

```python
class TextGenerator(Protocol):
    async def generate(self, req: TextRequest) -> TextResult: ...
class StructuredGenerator(Protocol):
    async def generate(self, req: StructuredRequest) -> StructuredResult: ...   # JSON schema-bound
class VisionAnalyzer(Protocol): ...
class OCRProvider(Protocol): ...
class HandwritingProvider(Protocol): ...
class Translator(Protocol): ...
class Embedder(Protocol): ...
class Reranker(Protocol): ...
class Reasoner(Protocol): ...
```

Each result is a `pydantic` model carrying: `value`, `raw`, `usage`, `latency_ms`,
`confidence`, `provider`, `model_id`, `model_version`, `error?` — the standard AI envelope
(`AI_ARCHITECTURE.md` §2.1).

## 3. Gateway flow

```
caller → router.select(task, ctx)          # MODEL_ROUTING
       → gateway.dispatch(capability, req)
            → provider adapter (from registry row)
            → retry policy (another adapter/model if eligible & failure transient)
            → model_runs + ai_usage + audit recorded (idempotent)
```

## 4. Provider adapters (MVP plan)

| Provider | Modes | Capabilities (initial) |
|---|---|---|
| OpenAI | cloud/paid | text, structured, vision, embeddings, translation |
| Gemini | cloud/paid | text, structured, vision, embeddings |
| Anthropic | cloud/paid | text, structured, vision, reasoning |
| GLM (Zhipu) | cloud/paid | text, structured, vision |
| Ollama | local/free | text, structured, vision, embeddings, rerank (via local models) |
| vLLM | self-hosted | text, structured, embeddings (OpenAI-compatible server) |
| Local Tesseract | local/free | OCR (printed EN/UR) |
| Local PaddleOCR | local/free | OCR (printed; handwriting batch) |
| VLM adapter (e.g., Qwen-VL/GPT-4o-mini) | local/cloud | OCR, handwriting consensus member |
| Local bge-m3 (ONNX/vLLM) | local/free | embeddings (+ sparse) |
| bge-reranker (local) | local/free | rerank |

Adapters are thin, versioned, and testable with recorded fixtures (no network needed in
unit tests). New providers = new adapter + registry row — no business-logic changes.

## 5. Key design points

- **Unified content types**: images sent as refs or inline-bytes with size guardrails;
  token accounting normalized to the provider's unit then converted to a canonical ledger
  (`model_runs`).
- **Structured output**: prefer provider JSON mode/schemas; otherwise JSON repair + schema
  validation with retry (`StructuredResult.validated` flag).
- **Timeouts & backpressure**: per-task timeout config; concurrency limiter per provider;
  queue depth metrics ([QUEUES_WORKERS.md](QUEUES_WORKERS.md)).
- **Secrets**: keys from env/secret manager, never in code/repo
  ([SECURITY.md](SECURITY.md)); per-provider key rotation supported.
- **Credential-less local mode**: everything can run with Ollama + Tesseract + PaddleOCR +
  bge-m3/bge-reranker locally — a fully offline school deployment is possible (open
  question #4; see DEPLOYMENT.md).

## 6. Failure semantics

- Transport/timeout/rate-limit errors → classified transient → router may retry on another
  provider with the same capability and results-compatible model; both attempts logged.
- Schema-violating completions → repair up to N attempts → fail job (retryable) rather than
  persist malformed candidates.
- No silent fallback across providers with different behavior without logging the switch.

## 7. Testing & evaluation

- Adapter contract tests against recorded provider responses (golden fixtures).
- Eval lab runs the same adapter set against golden datasets to compare providers
  (`AI_EVALUATION.md`), informing routing defaults.