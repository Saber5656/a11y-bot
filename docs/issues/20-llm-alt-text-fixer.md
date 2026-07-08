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

1. Image resolution (alt cases): resolve the `src` value to a **local file**:
   - static string relative path → resolve against the HTML/SFC/JSX file's dir,
     then against configured web roots (`scan` include roots; document order);
   - imported asset identifier (JSX `import img from "./x.png"`) → follow the
     import's literal path only;
   - remote URLs, dynamic expressions, data URIs, unresolvable paths → plan
     null → `needs_human` (v1 never fetches remote images — DESIGN §10.2.4).
   - Supported formats png/jpe?g/gif/webp/svg (svg passed as text, ≤ 16 KiB);
     rasters downscaled to max 1024 px (sharp is NOT added — use Playwright's
     chromium? No: add `jimp`? — implementer note: use the lightweight
     `@napi-rs/canvas`-free approach: pass original bytes if ≤ 1 MiB and
     dimensions unknown; else skip with `needs_human` note `image-too-large`.
     Record the chosen approach in code comment; no native deps allowed.)
2. Prompt: system preamble (issue 19) + task instructions; user content =
   framed untrusted blocks for (a) surrounding markup snippet (±3 elements),
   (b) page/file path context, plus the image part. JSON schema:
   `{ alt: string (1..150), decorative: boolean, confidence: 0..1 }`.
3. Result handling:
   - `decorative: true` AND confidence ≥ 0.8 → plan `alt=""` with
     `decorativeConfirmed: true`, class **auto_review** override (engine
     policy from issue 13);
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
      static), decorative path (auto_review + `alt=""` guard satisfied),
      low-confidence → needs_human, validator-reject → needs_human.
- [ ] Image resolution matrix tested (relative, imported, remote→skip,
      data URI→skip, missing file→skip, oversized→skip).
- [ ] iframe title path tested (HTML + JSX).
- [ ] Attribute escaping test (`alt` containing quotes/ampersands renders valid
      markup and passes re-lint verify).
- [ ] End-to-end via `fix --llm` with stub base URL: PR-body summary marks the
      fix AI-generated (integration test through issue 17 command).

## Validation

```bash
npm test -- src/llm/fixers
A11YBOT_LLM_API_KEY=stub node dist/cli/index.js fix fixtures/static-html --llm --dry-run   # with stub server per test harness docs
```

## Dependencies

19, 13.

## Non-goals

aria-label suggestions for icon buttons (fold into v2 unless trivially similar —
explicitly deferred); analyst (28); remote image fetching.

## Design References

DESIGN.md §10.3, §10.2, §9.1 (empty-alt guard), §14.2 T1/T2.
