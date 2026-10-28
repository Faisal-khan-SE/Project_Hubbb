# ADR 0002: AI in a separate Python service, behind ports, with a mock mode

- **Status:** Accepted
- **Date:** 2026-10-28
- **Deciders:** Founder A, Founder B

## Context
The platform uses AI for three things: project price/time estimates, layer-2 moderation (off-platform
intent), and a dashboard chatbot. LLM providers can be slow, down, expensive or change. Backend and frontend
work must not be blocked waiting for AI, and tests must be deterministic.

## Decision
- A separate FastAPI service (`ai-service/`, Python 3.11) exposes `/v1/estimate`, `/v1/moderate`, `/v1/chat`,
  `/health`, defined in `contracts/openapi/ai-service.yaml`. The backend calls it over HTTP with header
  `X-Internal-Api-Key` (D9).
- In the backend, AI is reached only through `AiModelPort` / `AiModerationPort` adapters, with fakes for tests.
- `AI_MODE=mock` returns deterministic responses and makes no network calls; used in dev, CI and e2e.
- Strict timeouts and fallbacks: estimate runs as a background job (20 s, retries, then `estimate_failed`);
  moderation has an 800 ms budget and falls back to layer-1 rules (D8); chat falls back to a fixed message.
- Fees, penalties, escrow and auth decisions are NEVER made by AI; AI only suggests (estimate) or scores (moderation).
- The LLM provider is behind an `LlmClient` interface in the Python service and has a monthly cost guard.

## Alternatives considered
- **Call the LLM directly from Java:** fewer services, but Python has much better tooling for evaluation and prompts, and mixing it into the monolith makes testing harder.
- **No AI for estimates (manual bands only):** simpler, but loses a core product promise.
- **Self-hosted model:** no per-call cost, but needs GPU hosting and ML ops the team does not have yet.

## Consequences
- Two deployables instead of one; the contract file must be kept in sync and reviewed by both founders.
- The platform keeps working (degraded) when AI is down.
- Mock mode lets backend and frontend progress in parallel with AI work.
