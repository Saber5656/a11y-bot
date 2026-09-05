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
  `src/llm/budget.ts`. (`llm.capabilities` already exists in the schema —
  issue 02 implements all §6.2 keys; this issue consumes it.)
- Budget summary: `budget.getSummary()` returns `{ calls, inputTokens,
  outputTokens, rejectedOutputs }`; commands merge it into the report envelope
  as `run.llm` (optional field defined by issue 09's envelope schema).

## Detailed Requirements

1. Interface exactly per DESIGN §10.1 (`capabilities()`, `complete()` with
   `TextPart | ImagePart`, optional `jsonSchema`, `maxOutputTokens`).
2. Factory `createLlmClient(config, env, opts: { enabled: boolean }):
   LlmClient | null`:
   - `opts.enabled` is the EXPLICIT feature gate (DESIGN §14.3: LLM off unless
     key AND explicit config/flag) — callers pass `fix.llm`/`--llm` (fix path)
     or the audit analyst enablement; `enabled: false` → null regardless of key.
   - returns `null` when no key present in `env[config.llm.apiKeyEnv]` (fallback
     `OPENAI_API_KEY`) — callers treat null as "feature disabled", logging one
     info line per feature (`skipped (no LLM key)`).
   - never throws for missing key; throws ConfigError for malformed baseUrl.
3. OpenAI-compatible provider:
   - POST `{baseUrl}/chat/completions`; payload: model, temperature 0,
     **primary token field `max_tokens`**; if the endpoint responds 400 with an
     error message naming that field, retry once with
     `max_completion_tokens` (fallback flagged in result meta). Image parts:
     `{ type: "image_url", image_url: { url:
     "data:<mediaType>;base64,<base64>" } }` (mediaType from `ImagePart`).
   - `jsonSchema` given → `response_format: { type: "json_schema", json_schema:
     { name: "a11ybot_output", schema, strict: true } }` (fixed name constant);
     on 400 for unsupported response_format, retry once with
     instruction-embedded JSON + local parse (fallback path flagged in result
     meta).
   - Timeout via AbortController (`llm.timeoutMs`); retries: ≤ 2 on 429/5xx with
     exponential backoff (1 s, 4 s) honoring `Retry-After`.
   - Errors surface as `LlmUnavailableError` — a module-local class exported
     from `src/llm/client.ts`, deliberately OUTSIDE the core exit-code taxonomy
     because features catch it and degrade to skip (never crashes the run;
     EnvError only if the user explicitly forced `--llm` AND every call failed —
     decided at command level, not here).
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
      path, `max_tokens`→`max_completion_tokens` 400-fallback path, 429 retry
      honoring `Retry-After` (delay asserted with fake timers), timeout abort,
      500→LlmUnavailableError, image part encoding per mediaType.
- [ ] Gating matrix tested: (enabled,key) = (false,set)→null, (true,unset)→null,
      (true,set)→client, (false,unset)→null.
- [ ] Capabilities override consumed correctly: `{ vision: false }` config
      makes `capabilities().vision === false` regardless of provider defaults.
- [ ] `getSummary()` totals verified across multiple calls.
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
