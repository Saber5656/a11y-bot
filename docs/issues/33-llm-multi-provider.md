# Title

Multi-provider LLM SDK layer (provider registry, capability flags)

## Summary

Extract a provider registry behind the `LlmClient` interface and add official-SDK
providers, keeping `openai-compatible` as the default.

## Context

Owner requirement (2026-07-08): MVP ships OpenAI-compatible only; multi-provider
SDK support must exist as planned v1 issues. ADR-004.3. All guardrails (19) and
features (20/28) are provider-agnostic already; this issue must not touch them.

## Scope

- `src/llm/providers/` registry refactor + new providers; config extension.

## Detailed Requirements

1. Registry: `llm.provider` value → factory map:
   - `openai-compatible` (existing, default — behavior unchanged; regression
     suite must stay green untouched);
   - `openai` (official `openai` SDK; supports `response_format` json_schema,
     vision; key env default `OPENAI_API_KEY`);
   - `google` (official `@google/genai` SDK; schema-constrained output via its
     structured output mode; vision; key env default `GEMINI_API_KEY`).
   Each provider module declares `capabilities()` statically; feature code
   already consults capabilities (18) — verify and add missing checks
   (no-vision provider + alt-text fixer → needs_human with note
   `provider-no-vision`, tested).
2. Config: `llm.provider` enum extended; per-provider optional `llm.model`
   defaults table (constant in code, documented in config guide);
   `llm.apiKeyEnv` still wins when set. Unknown provider value → ConfigError
   listing supported ids.
3. SDK dependencies are **optional peerDependencies**? — NO: keep them regular
   dependencies ONLY if combined install size increase < 15 MB; otherwise use
   lazy dynamic `import()` with a clear EnvError when the SDK is absent
   (choose, measure, record decision + numbers in an ADR-009 added in this
   PR). This decision affects OSS install ergonomics — surface the measurement
   in the PR description.
4. Provider conformance suite: one shared vitest suite runs against every
   provider with its SDK mocked at the HTTP layer where feasible (openai SDK
   supports custom fetch; google SDK per its test guidance): happy path,
   json-schema path, timeout, retry/backoff, capability flags, key-missing →
   null client. Adversarial corpus (19) rerun against each provider's parse
   path.
5. Docs: `docs/guides/llm-providers.md` — setup per provider (key env, model
   examples, local LLM via openai-compatible baseUrl), determinism/cost notes.

## Acceptance Criteria

- [ ] Conformance suite green for all three providers; openai-compatible
      snapshots unchanged (regression guard).
- [ ] Capability-gating test: google/openai vision on; a mock no-vision
      provider exercises the needs_human path.
- [ ] ADR-009 records the dependency-strategy decision with measured sizes.
- [ ] `fix --llm` and `audit --llm` run end-to-end against stub servers for
      each provider (integration matrix).
- [ ] Guide published; config schema + JSON Schema regenerated.

## Validation

```bash
npm test -- src/llm/providers
npm pack --dry-run   # size check recorded in PR
```

## Dependencies

18, 19, 20, 28.

## Non-goals

Anthropic provider (workflow policy; openai-compatible endpoints remain usable
for any user-configured host); provider auto-detection; streaming.

## Design References

DESIGN.md §10.1; ADR-004; research/2026-07-llm-providers.md.
