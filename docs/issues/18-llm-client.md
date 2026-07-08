# Title

LLM client core: OpenAI-compatible provider, budget, config

## Summary

Implement the optional LLM client used by content fixers and the UX analyst: an
OpenAI-compatible HTTP provider, key resolution, run budget, and the disabled
(no-key) mode.

## Context

ADR-004 and DESIGN.md §10.1 are normative. Research doc `2026-07-llm-providers.md`
sets the default model (`gpt-5.4-mini`, vision support to be confirmed — known
unknown U1). No SDK dependencies in this issue (fetch only); provider registry
extraction happens in issue 33.

## Scope

- `src/llm/client.ts` (interface + factory), `src/llm/providers/openai-compatible.ts`,
  `src/llm/budget.ts`.

## Detailed Requirements

1. Interface exactly per DESIGN §10.1 (`capabilities()`, `complete()` with
   `TextPart | ImagePart`, optional `jsonSchema`, `maxOutputTokens`).
2. Factory `createLlmClient(config, env): LlmClient | null`:
   - returns `null` when no key present in `env[config.llm.apiKeyEnv]` (fallback
     `OPENAI_API_KEY`) — callers treat null as "feature disabled", logging one
     info line per feature (`skipped (no LLM key)`).
   - never throws for missing key; throws ConfigError for malformed baseUrl.
3. OpenAI-compatible provider:
   - POST `{baseUrl}/chat/completions`; payload: model, temperature 0,
     `max_tokens` (or `max_completion_tokens` — send whichever the endpoint
     accepts; implement primary + one retry with the alternate field name on a
     400 naming that field), messages with image parts as
     `{ type: "image_url", image_url: { url: "data:image/png;base64,…" } }`.
   - `jsonSchema` given → `response_format: { type: "json_schema", json_schema:
     { name, schema, strict: true } }`; on 400 for unsupported response_format,
     retry once with instruction-embedded JSON + local parse (fallback path
     flagged in result meta).
   - Timeout via AbortController (`llm.timeoutMs`); retries: ≤ 2 on 429/5xx with
     exponential backoff (1 s, 4 s) honoring `Retry-After`.
   - Errors surface as `LlmUnavailable` (feature-level skip + report notice,
     never crashes the run; EnvError only if user explicitly forced `--llm`
     AND every call failed — decided at command level, not here).
   - `capabilities()`: `{ vision: true, structuredOutput: true }` for the
     default provider, overridable via config `llm.capabilities` (escape hatch
     for non-vision compatible endpoints; U1 verification recorded here: confirm
     default-model vision via a documented manual smoke test and pin result in
     code comment + DESIGN §2.3 update).
4. Budget (`budget.ts`): per-run counters (calls, input/output tokens from
   usage fields); `tryReserveCall()` returns false once `llm.maxCalls` reached →
   caller skips with report notice; token totals appear in run summary.
5. Security: request/response bodies logged only at debug level AFTER redaction
   (issue 04 logger); key never appears in errors (test asserts).

## Acceptance Criteria

- [ ] Stub-server tests (local HTTP): happy path, json_schema path + fallback
      path, 429 retry, timeout abort, 500→LlmUnavailable, image part encoding.
- [ ] No-key mode returns null client; feature-skip logging verified in 20/28
      (here: unit test on factory).
- [ ] Budget exhaustion test: maxCalls=2, third call skipped.
- [ ] Redaction test: forced error containing the key string logs `***`.
- [ ] Zero LLM dependencies added to package.json (fetch only).

## Validation

```bash
npm test -- src/llm
```

## Dependencies

02, 04.

## Non-goals

Prompts/guardrails (19), features (20/28), SDK providers (33).

## Design References

DESIGN.md §10.1, §2.3 U1, §14.2 T8; ADR-004; research/2026-07-llm-providers.md.
