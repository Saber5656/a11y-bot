# Title

Deterministic fixer batch: JSX rules

## Summary

Implement deterministic fixers for jsx-a11y rules classified auto_safe /
auto_review in issue 06, operating on the estree/JSX AST spans.

## Context

Engine contract from issue 13 (`SourceFile` in, `FixPlan` out; fixers re-parse
via the same `typescript-eslint` parser pinned in issue 05). Editing principle
(normative): **attribute REMOVAL fixers act on any value form** (string,
expression, bare) since removal is value-independent; **value-MODIFYING fixers
act only on exact literal cases** (`alt={expr}` etc. → plan null →
`fix_skipped`), keeping v1 free of semantic JS rewriting. Rule ids below are
shorthand for `static/jsx-a11y/<rule>`.

## Scope

- `src/fix/fixers/jsx/*.ts`, registration entries in `src/fix/catalog.ts`,
  `fix.note` catalog messages, golden fixtures under `fixtures/fixers/jsx/`.

## Detailed Requirements

Normative fixer set:

| # | ruleId (static/jsx-a11y/…) | Class | Transformation |
|---|---|---|---|
| 1 | `no-access-key` | auto_safe | remove the `accessKey` attribute — any value form (removal fixer) |
| 2 | `no-redundant-roles` | auto_safe | remove the reported redundant `role` attribute for ANY element/role pair the upstream rule flags (e.g. `<img role="img" />` → `<img />`); string-literal role values only — expression roles → null |
| 3 | `no-autofocus` | auto_review | remove `autoFocus` attribute — any value form |
| 4 | `tabindex-no-positive` | auto_review | `tabIndex={3}` / `tabIndex="3"` → `0` (numeric literal or string literal only; other expressions → null) |
| 5 | `aria-props` | auto_review | rename misspelled `aria-*` attribute to the unique nearest valid name (Levenshtein distance ≤ 2 against the WAI-ARIA 1.2 property list vendored as a constant); no unique candidate → null |
| 6 | `img-redundant-alt` | auto_review | strip leading/trailing redundant words from **string-literal** alt values: patterns `/\b(image|picture|photo|photograph|graphic)( of)?\b/i` with whitespace cleanup; empty result after strip → null |
| 7 | `html-has-lang` / `lang` | auto_safe (config-gated) | `<html>` → e.g. `<html lang="en">` (string-literal attribute rendered from the configured `fix.defaults.lang` value), only when configured and existing value absent/empty-literal |

Shared requirements:

1. `allowedSpan` = JSX opening element span (`FixPlan.allowedSpan`, issue 13).
2. Value-modifying fixers: expression containers other than the exact literal
   cases above are never edited (tests assert `fix_skipped`); removal fixers
   are exempt per the Context principle.
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
- [ ] Removal-vs-modify principle tested: `accessKey={expr}` removed;
      `alt={expr}`-style value edits skipped; spread-attribute skip per fixer.
- [ ] `aria-props` renames `aria-lable`→`aria-label`; ambiguous typo case
      returns null (both tested).
- [ ] Class policy: auto_review fixers inert under default `fix.classes`.
- [ ] Security (DESIGN §14.2 T2): mutated-fixer negative tests — out-of-span
      edit and `on*=`/script-bearing newText rejected by the engine validators.
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

DESIGN.md §9.1–9.3, §14.2 T2; issue 06 fixability table.
