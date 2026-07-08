# ADR-004: LLM features are optional; OpenAI-compatible client first, SDK multi-provider later

- Status: accepted (2026-07-08, approved by product owner)

## Context

Two features benefit from an LLM: content-required fixes (e.g., generating alt
text — needs vision) and UX analysis of runtime evidence. Requiring an API key for
core value would hurt OSS adoption; excluding LLMs entirely would cap the product at
placeholder-quality fixes. Research: [2026-07-llm-providers](../research/2026-07-llm-providers.md).

## Decision

1. **No-key-required baseline**: every command works with zero LLM configuration;
   LLM-dependent steps are skipped with an explicit notice.
2. MVP client: a single fetch-based **OpenAI-compatible** client
   (`baseUrl` + `model` + key-bearing env var), default model `gpt-5.4-mini`
   (vision support to be confirmed at implementation; see known unknowns).
3. Multi-provider **SDK** support (capability-flagged provider interface, official
   SDKs) is planned as explicit issues in a later wave (owner requirement), not as
   v1-blocking work.
4. Hard guards regardless of provider: per-run call/token caps, `temperature: 0`,
   schema-constrained outputs, and re-validation of all outputs (re-lint gate for
   patches). LLM output never gates CI.

## Consequences

- OSS users without keys get the full deterministic product.
- Local/self-hosted LLMs work day one via `baseUrl` override.
- Provider proliferation cost is deferred and isolated behind one interface.
