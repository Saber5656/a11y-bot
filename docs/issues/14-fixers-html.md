# Title

Deterministic fixer batch: HTML rules

## Summary

Implement the deterministic fixers for the HTML adapter rules classified
auto_safe / auto_review in issue 08, with golden before/after tests.

## Context

Fixers receive the finding + parsed source (via `@html-eslint/parser` AST from
the lint pass) and emit `FixPlan`s under the engine's policy guardrails
(issue 13). Positions come from the AST — no regex-on-HTML editing.

## Scope

- `src/fix/fixers/html/*.ts` — one module per fixer, registered in catalog.

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
| 7 | `require-lang` | auto_safe (config-gated) | `<html>` → `<html lang="{fix.defaults.lang}">` only when configured; plan returns null otherwise |
| 8 | `no-non-scalable-viewport` | auto_review | in the viewport `content`, remove `user-scalable=no` and any `maximum-scale<5`, normalizing separators (`<meta name="viewport" content="width=device-width, user-scalable=no">` → `…content="width=device-width"`) |

Shared requirements:

1. `allowedSpan` = the owning element's start-tag span (engine validates).
2. Attribute removal must remove exactly one attribute and one adjacent
   whitespace run; resulting tag re-serializes to valid HTML (verified by
   re-parse in the engine loop).
3. Each fixer ships golden fixtures: `fixtures/fixers/html/<rule>/{input.html,
   expected.html}` (multiple cases per rule where behavior branches: quoted /
   unquoted / single-quoted attribute values; attribute order variations).
4. Message-catalog `fix.note` entries per fixer (English, one line) for PR body.
5. Content-required rules (`require-img-alt`, `require-frame-title`,
   `require-input-label`) are explicitly NOT implemented here (issue 20).

## Acceptance Criteria

- [ ] All 8 fixers registered; golden tests pass byte-exact.
- [ ] Engine verify loop green for every golden case (fix removes finding, adds
      none) — asserted by an integration test driving engine + HTML profile.
- [ ] Double-run idempotence on golden inputs.
- [ ] Attribute-value quoting styles preserved (tests cover `'` / `"` / bare).
- [ ] Viewport fixer handles multi-directive content values incl. spaces.

## Validation

```bash
npm test -- src/fix/fixers/html
```

## Dependencies

13, 08.

## Non-goals

LLM content fixes (20); JSX/Vue fixers (15/16).

## Design References

DESIGN.md §9.3 (catalog rows), §9.1 (classes), §16 (golden tests).
