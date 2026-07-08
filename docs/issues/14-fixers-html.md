# Title

Deterministic fixer batch: HTML rules

## Summary

Implement the deterministic fixers for the HTML adapter rules classified
auto_safe / auto_review in issue 08, with golden before/after tests.

## Context

Fixers receive the finding + `SourceFile` (issue 13) and emit `FixPlan`s under
the engine's policy guardrails. `ESLint.LintResult` does NOT expose the AST, so
each fixer re-parses `SourceFile.text` itself with `@html-eslint/parser`'s
`parseForESLint()` to obtain node offsets — no regex-on-HTML editing. Rule ids
below are shorthand for the unified ids `static/html-eslint/<rule>` (registry
and catalog keys always use the unified form).

## Scope

- `src/fix/fixers/html/*.ts` — one module per fixer.
- Registration entries in `src/fix/catalog.ts` (issue 13's registry).
- `fix.note` message-catalog entries in `src/core/messages/en.ts`.
- Golden fixtures under `fixtures/fixers/html/`.

## Detailed Requirements

Implement exactly these fixers (normative spec; before → after):

| # | ruleId (static/html-eslint/…) | Class | Transformation |
|---|---|---|---|
| 1 | `no-redundant-role` | auto_safe | `<img role="img">` → `<img>` (remove the `role` attribute node incl. surrounding whitespace normalization) |
| 2 | `no-aria-hidden-body` | auto_safe | `<body aria-hidden="true">` → `<body>` |
| 3 | `no-accesskey-attrs` | auto_safe | remove `accesskey="…"` attribute |
| 4 | `no-positive-tabindex` | auto_review | `tabindex="3"` → `tabindex="0"` (value edit only) |
| 5 | `no-abstract-roles` / `no-invalid-role` | auto_review | remove the `role` attribute |
| 6 | `no-aria-hidden-on-focusable` | auto_review | remove `aria-hidden` attribute from the focusable element |
| 7 | `require-lang` | auto_safe (config-gated) | `<html>` → e.g. `<html lang="en">` from `fix.defaults.lang`; plan returns null when unset. The config value is schema-validated as a BCP-47-shaped tag (`^[a-zA-Z]{2,3}(-[a-zA-Z0-9]{2,8})*$`, added to issue 02's schema by this issue) and attribute-escaped on insertion |
| 8 | `no-non-scalable-viewport` | auto_review | see viewport spec below |

Viewport fixer spec (normative): parse `content` by splitting on `,` and `;`
(both legal separators in the wild), trim each directive, compare names
case-insensitively; drop every `user-scalable=no` directive and every
`maximum-scale=<n>` with n < 5 (invalid/non-numeric n → directive kept,
plan proceeds on other matches); duplicates all dropped; remaining directives
re-joined with `", "` preserving original order; empty result → remove the
`content` attribute value edit and return null (nothing safe to write).

Shared requirements:

1. `allowedSpan` = the owning element's start-tag span, computed from the
   parsed AST node offsets (`FixPlan.allowedSpan` per issue 13).
2. Attribute removal must remove exactly one attribute and one adjacent
   whitespace run; resulting tag re-serializes to valid HTML (verified by
   re-parse in the engine loop).
3. Each fixer ships golden fixtures:
   `fixtures/fixers/html/<rule>/<case>/{input.html,expected.html}` — one
   directory per case (`quoted`, `single-quoted`, `unquoted`, `multi-attr`,
   plus rule-specific branches).
4. Message-catalog `fix.note` entries per fixer (English, one line) for PR body.
5. Content-required rules (`require-img-alt`, `require-frame-title`,
   `require-input-label`) are explicitly NOT implemented here (issue 20).

## Acceptance Criteria

- [ ] All 8 fixers registered; golden tests pass byte-exact.
- [ ] Engine verify loop green for every golden case (fix removes finding, adds
      none) — asserted by an integration test driving engine + HTML profile.
- [ ] Class policy verified end-to-end: with default `fix.classes:
      [auto_safe]`, auto_review fixers (4/5/6/8) do NOT apply; they apply once
      `auto_review` is enabled (both directions tested).
- [ ] Security (DESIGN §14.2 T2 / §9.1): a hostile `fix.defaults.lang` value
      (`en" onload="x`) is rejected by schema validation; a mutated fixer
      emitting an out-of-span or `on*=`-bearing edit is rejected by the engine
      (negative tests through `patch-validators`).
- [ ] Double-run idempotence on golden inputs.
- [ ] Attribute-value quoting styles preserved (tests cover `'` / `"` / bare).
- [ ] Viewport fixer covers: multi-directive, `;` separators, uppercase names,
      duplicates, invalid numeric maximum-scale, empty-result null.

## Validation

```bash
npm test -- src/fix/fixers/html
```

## Dependencies

13, 08.

## Non-goals

LLM content fixes (20); JSX/Vue fixers (15/16).

## Design References

DESIGN.md §9.1 (classes, guardrails), §9.2 (allowedSpan/engine contract),
§9.3 (catalog rows), §14.2 T2, §16 (golden tests).
