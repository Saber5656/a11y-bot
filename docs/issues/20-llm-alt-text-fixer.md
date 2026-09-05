# Title

LLM content fixer: alt text / frame title generation

## Summary

Implement the `content_required` fixer that generates image alt text (vision)
and iframe/frame titles via the guarded LLM client, plugged into the fix engine
as class `content_required`.

## Context

DESIGN.md §10.3. Covers rules mapped `content_required` in issues 06–08:
`static/jsx-a11y/alt-text`, `static/html-eslint/require-img-alt`,
`static/vuejs-a11y/alt-text` (img cases), `static/html-eslint/require-frame-title`,
`static/jsx-a11y/iframe-has-title` (title from context, no vision),
`static/vuejs-a11y/iframe-has-title`. Runs only with `fix.llm: true` (or
`--llm`) AND a key (client non-null); otherwise engine reports `needs_human`.

## Scope

- `src/llm/fixers/alt-text.ts` (+ registration), image resolution helper.

## Detailed Requirements

1. Image resolution (alt cases): resolve the `src` value to a **local file**.
   Resolution order (normative): (a) relative to the directory of the file
   containing the finding; (b) relative to the repo root; (c) relative to each
   configured `audit.targets[].staticDir` in config order. First existing file
   wins.
   - imported asset identifier (JSX `import img from "./x.png"`) → follow the
     import's literal path only;
   - remote URLs, dynamic expressions, data URIs, unresolvable paths → the
     finding is reported `needs_human` (v1 never fetches remote images —
     DESIGN §10.2.4).
   - Supported inputs: png/jpe?g/webp/gif rasters passed as `ImagePart`
     `{ base64, mediaType }` (DESIGN §10.1) when file size ≤ 1 MiB — **no
     image-processing library, no native deps; bytes pass through unmodified**;
     larger files → `needs_human` note `image-too-large`. SVG is passed as
     framed TEXT (≤ 16 KiB, larger → `needs_human`).
2. Prompt: system preamble (issue 19) + task instructions; user content =
   framed untrusted blocks for (a) surrounding markup snippet (±3 elements),
   (b) repo-relative file path context (`assertPromptClean` enforced), plus the
   image part. JSON schema (matches DESIGN §10.3):
   `{ alt: string (1..150), decorative: boolean, confidence: 0..1 }`.
3. Result handling:
   - `decorative: true` AND confidence ≥ 0.8 → plan `alt=""` with
     `decorativeConfirmed: true`, treated as class **auto_review** (engine
     policy, issue 13) — therefore applied ONLY when `auto_review` ∈
     `fix.classes`; otherwise reported `needs_human` with note
     `decorative-requires-auto-review`;
   - else `alt="<validated text>"` (validatePatchText, attribute-escaped:
     `"` → `&quot;`), class content_required;
   - confidence < 0.5 or validator rejection → null (`needs_human`, reason
     recorded).
4. Frame/iframe title: no image; context = framed src URL string + surrounding
   markup; schema `{ title: string (1..80), confidence }`; same thresholds.
5. Every produced plan sets `llmGenerated: true` (feeds PR-body AI section and
   `[llm]` commit marker). Budget consumed per finding; budget exhausted →
   remaining findings `needs_human` with note `llm-budget-exhausted`.
6. Determinism note (documented in code + report): temperature 0 but outputs
   may vary between runs; fingerprint of the finding, not the text, keys
   idempotence — a previously fixed finding no longer appears, so re-runs do
   not regenerate.

## Acceptance Criteria

- [ ] Stub-provider tests: happy alt path (JSX string literal, HTML, Vue
      static), decorative path (auto_review enabled vs not — both directions),
      low-confidence → needs_human, validator-reject → needs_human.
- [ ] Image resolution matrix tested (file-dir relative, repo-root relative,
      staticDir fallback, imported, remote→needs_human, data URI→needs_human,
      missing→needs_human, oversized→needs_human, svg-as-text path).
- [ ] iframe title path tested (HTML + JSX).
- [ ] Attribute escaping test (`alt` containing quotes/ampersands renders valid
      markup and passes re-lint verify).
- [ ] Security (DESIGN §14.2 T1/T2, §10.2): adversarial-corpus reuse — hostile
      stub outputs (tag smuggling, URL, oversize, sentinel escape) all land in
      needs_human; assembled prompts pass `assertPromptClean` with planted env
      canaries; an out-of-range stub-driven plan is rejected by the engine.
- [ ] End-to-end via `fix --llm --dry-run` with the stub server: the issue-17
      JSON summary marks the entry `llmGenerated: true` and the markdown
      summary renders it under the AI-generated section (constants from
      issue 19).

## Validation

```bash
npm test -- src/llm/fixers
# stub harness (test/helpers/llm-stub.ts, started by the npm script below on
# an ephemeral port and injected via a generated temp config with
# llm.baseUrl=http://127.0.0.1:<port>/v1):
npm run test:llm-stub-e2e
```

## Dependencies

19, 13.

## Non-goals

aria-label suggestions for icon buttons (fold into v2 unless trivially similar —
explicitly deferred); analyst (28); remote image fetching.

## Design References

DESIGN.md §10.1 (client interface, ImagePart, budget), §10.2, §10.3, §9.1
(empty-alt guard, class gating), §14.2 T1/T2.
