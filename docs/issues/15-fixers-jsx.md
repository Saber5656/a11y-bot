# Title

Deterministic fixer batch: JSX rules

## Summary

Implement deterministic fixers for jsx-a11y rules classified auto_safe /
auto_review in issue 06, operating on the estree/JSX AST spans.

## Context

Same engine contract as issue 14. JSX specifics: attributes are
`JSXAttribute` nodes; expression-valued attributes (`alt={expr}`) must be left
untouched by text-level fixers (plan → null, `fix_skipped`), keeping v1 free of
semantic JS rewriting.

## Scope

- `src/fix/fixers/jsx/*.ts` + golden fixtures.

## Detailed Requirements

Normative fixer set:

| # | ruleId (static/jsx-a11y/…) | Class | Transformation |
|---|---|---|---|
| 1 | `no-access-key` | auto_safe | remove `accessKey={…}`/`accessKey="…"` attribute |
| 2 | `no-redundant-roles` | auto_safe | `<img role="img" />` → `<img />` |
| 3 | `no-autofocus` | auto_review | remove `autoFocus` attribute |
| 4 | `tabindex-no-positive` | auto_review | `tabIndex={3}` / `tabIndex="3"` → `0` (numeric literal or string literal only; other expressions → null) |
| 5 | `aria-props` | auto_review | rename misspelled `aria-*` attribute to the unique nearest valid name (Levenshtein distance ≤ 2 against the WAI-ARIA 1.2 property list vendored as a constant); no unique candidate → null |
| 6 | `img-redundant-alt` | auto_review | strip leading/trailing redundant words from **string-literal** alt values: patterns `/\b(image|picture|photo|photograph|graphic)( of)?\b/i` with whitespace cleanup; empty result after strip → null |
| 7 | `html-has-lang` / `lang` | auto_safe (config-gated) | `<html>` → `<html lang="{fix.defaults.lang}">` (string literal), only when configured and existing value absent/empty-literal |

Shared requirements:

1. `allowedSpan` = JSX opening element span.
2. Expression containers (`{...}`) other than the exact literal cases above are
   never edited (tests assert `fix_skipped`).
3. Spread attributes on the element (`{...props}`) downgrade all fixers on that
   element to null (cannot prove attribute absence/uniqueness) — engine reports
   `fix_skipped` with note `spread-attributes`.
4. Golden fixtures under `fixtures/fixers/jsx/<rule>/` incl. `.tsx` cases and
   self-closing/multiline tags.
5. The vendored ARIA property list is a single constant module
   (`src/fix/fixers/jsx/aria-props-list.ts`) with source URL comment; shared
   with issue 16.

## Acceptance Criteria

- [ ] All 7 fixers registered; golden tests byte-exact; verify loop green.
- [ ] Expression-value and spread-attribute skip behaviors tested per fixer.
- [ ] `aria-props` renames `aria-lable`→`aria-label`; ambiguous typo case
      returns null (both tested).
- [ ] Idempotence double-run test.
- [ ] TSX parse path covered.

## Validation

```bash
npm test -- src/fix/fixers/jsx
```

## Dependencies

13, 06.

## Non-goals

alt-text generation (20); anchor/interaction-semantics rewrites (manual class).

## Design References

DESIGN.md §9.3, §9.1; issue 06 fixability table.
