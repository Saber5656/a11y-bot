# Title

Multi-provider LLM SDK layer (provider registry, capability flags)

## Summary

Extract a provider registry behind the `LlmClient` interface and add official-SDK
providers, keeping `openai-compatible` as the default.

## Context

Owner requirement (2026-07-08): MVP ships OpenAI-compatible only; multi-provider
SDK support must exist as planned v1 issues. ADR-004, Decision item 3.
Guardrails (19) stay untouched; features (20/28) receive exactly ONE addition
here — the vision-capability gate (they otherwise remain provider-agnostic).

## Scope

- `src/llm/providers/` registry refactor + new provider modules.
- `src/config/schema.ts`: extend the `llm.provider` enum; regenerate
  `schemas/a11ybot.schema.json`.
- `src/llm/capabilities.ts`: `requireVision(client)` helper; call sites added
  in `src/llm/fixers/alt-text.ts` (image attachment path) and
  `src/llm/analyst.ts` (screenshot attachment path) — no-vision providers →
  alt fixes become `needs_human` note `provider-no-vision`; analyst runs
  text-only evidence.
- `docs/guides/llm-providers.md` (new).

## Detailed Requirements

1. Registry: `llm.provider` value → factory map:
   - `openai-compatible` (existing, default — behavior unchanged; regression
     suite must stay green untouched);
   - `openai` (official `openai` SDK; `response_format` json_schema, vision);
   - `google` (official `@google/genai` SDK; schema-constrained output via its
     structured-output mode; vision).
   Each provider module declares `capabilities()` statically; the
   `requireVision` gate (Scope) is the single feature-side consumer added by
   this issue.
2. Config + defaults (normative):
   - `llm.provider` enum extended; unknown value → ConfigError listing
     supported ids.
   - Default model per provider (constant table, documented in the guide):
     `openai-compatible`/`openai` → `gpt-5.4-mini`; `google` →
     `gemini-2.5-flash` (verify the current stable id at implementation and
     record the source URL in the constant's comment).
   - Key env resolution (single rule, extends DESIGN §6.4): configured
     `llm.apiKeyEnv` ALWAYS wins; when that env var is unset, the provider's
     default env is consulted — `OPENAI_API_KEY` for openai/openai-compatible,
     `GEMINI_API_KEY` for google. Update DESIGN §6.4 accordingly in this PR.
3. SDK dependency strategy (decision procedure, recorded as ADR-009 in this
   PR): measure the `npm pack --dry-run` / install-size delta with both SDKs
   as regular dependencies; if the combined increase is < 15 MB → regular
   dependencies; otherwise → lazy dynamic `import()` with the SDKs left OUT of
   dependencies and an EnvError whose message contains the exact remedy
   (`npm install openai` / `npm install @google/genai`). Both branches are
   fully specified — the implementer measures, picks, and records numbers in
   ADR-009 + the PR description.
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
      provider exercises `provider-no-vision` needs_human (fixer) and
      text-only analyst paths.
- [ ] Security (T3/T8): conformance suite includes, per provider, (a) a
      planted-key SDK error whose surfaced message shows `***`, and (b)
      budget-cap + timeout enforcement (the shared budget wraps every
      provider — asserted via call counters).
- [ ] Adversarial corpus (issue 19) re-run against each provider's parse path.
- [ ] ADR-009 records the dependency-strategy decision with measured sizes;
      DESIGN §6.4 updated with the key-resolution rule.
- [ ] Guide published; config schema + JSON Schema regenerated.

## Validation

```bash
npm test -- src/llm/providers
npm run test:llm-stub-e2e -- --provider openai-compatible
npm run test:llm-stub-e2e -- --provider openai
npm run test:llm-stub-e2e -- --provider google
npm pack --dry-run   # size check recorded in PR + ADR-009
```

## Dependencies

18, 19, 20, 28.

## Non-goals

Anthropic provider (workflow policy; openai-compatible endpoints remain usable
for any user-configured host); provider auto-detection; streaming.

## Design References

DESIGN.md §10.1; ADR-004; research/2026-07-llm-providers.md.
