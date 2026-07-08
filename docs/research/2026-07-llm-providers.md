# Research: LLM provider landscape for optional features (2026-07)

Status: verified 2026-07-08 (web sources; pricing pages move fast — re-verify at
implementation time).
Decision consumer: [ADR-004](../decisions/ADR-004-llm-optional-openai-compatible-first.md), DESIGN.md §10.

## Question

Which provider strategy serves the two optional LLM features — content-required
fixes (alt text etc., needs **vision**) and evidence-bundle UX analysis (needs
vision + long context) — while keeping a11y-bot fully functional with no key?

## Verified facts (2026-07-08)

- OpenAI current mainstream production models (from developers.openai.com pricing):
  - `gpt-5.4-mini` — $0.75 / $4.50 per 1M tokens (in/out): default candidate.
  - `gpt-5.4-nano` — $0.20 / $1.25 per 1M tokens: cheaper, capability risk for
    nuanced a11y judgments.
  - Flagships (`gpt-5.5`, `gpt-5.6` family) are overkill for v1 defaults.
- The public pricing page does not explicitly tag vision support per model.
  **Open item for implementation:** confirm image-input support of the chosen
  default model id via the models endpoint / docs before hardcoding the default.
- The OpenAI-compatible REST surface (`/v1/chat/completions` and/or `/v1/responses`)
  is the industry lingua franca: local runtimes (Ollama, LM Studio, vLLM) and many
  hosted providers expose it. One HTTP client therefore covers OpenAI + most
  alternatives via `baseUrl` + `model` override.

## Strategy

| Phase | Support | Rationale |
|---|---|---|
| MVP (waves 3–4) | Single `openai-compatible` HTTP client (fetch-based, no SDK), configurable `baseUrl`/`model`/key env | Smallest dependency surface; covers OpenAI, local LLMs, compatible hosts |
| Wave 6 (issue 33) | Provider interface extraction + official SDK providers (e.g., OpenAI SDK, Google GenAI SDK) with capability flags (`vision`, `structuredOutput`) | User requirement: multi-provider SDK support planned as issues |

Anthropic/Claude is intentionally not a default (project workflow policy); the
provider interface must still allow any OpenAI-compatible or SDK-backed endpoint the
user configures.

## Cost & determinism guards (design inputs)

- Per-run caps: `llm.maxCalls` (default 20), `llm.maxInputTokensPerCall`,
  request timeout (default 60 s), total-run token budget.
- `temperature: 0` and JSON-schema-constrained outputs wherever supported.
- All LLM output is advisory or re-validated (re-lint gate for patches; schema
  validation for analyst findings). No LLM output ever gates CI by default.
- No key present → features log a one-line "skipped (no LLM key)" notice and the
  run continues deterministically. Exit code is unaffected.
