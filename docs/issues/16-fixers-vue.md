# Title

Deterministic fixer batch: Vue rules

## Summary

Implement deterministic fixers for vuejs-accessibility rules classified
auto_safe / auto_review in issue 07, operating on `vue-eslint-parser` template
AST spans.

## Context

Vue templates are HTML-like but attributes may be directive-bound
(`:alt="expr"`, `v-bind:tabindex`). Editing discipline (normative): attribute
VALUES are edited only in static-attribute form; attribute-NAME renames
(fixer 5) also apply to the bound form because the rename never touches the
value expression; removal fixers act on the static form only in Vue (bound
attributes may be conditionally meaningful). Known unknown U5 (ESLint autofix
quality in `.vue`) is resolved here by NOT using upstream autofixes at all —
all edits are our own attribute-level text edits computed from template AST
offsets (SFC-absolute). Rule ids below are shorthand for
`static/vuejs-a11y/<rule>`.

## Scope

- `src/fix/fixers/vue/*.ts`, registration entries in `src/fix/catalog.ts`,
  `fix.note` catalog messages, golden fixtures under `fixtures/fixers/vue/`.

## Detailed Requirements

Normative fixer set:

| # | ruleId (static/vuejs-a11y/…) | Class | Transformation |
|---|---|---|---|
| 1 | `no-access-key` | auto_safe | remove static `accesskey` attribute |
| 2 | `no-redundant-roles` | auto_safe | remove redundant static `role` |
| 3 | `no-autofocus` | auto_review | remove static `autofocus` |
| 4 | `tabindex-no-positive` | auto_review | static `tabindex="3"` → `tabindex="0"`; bound `:tabindex` → null |
| 5 | `aria-props` | auto_review | rename misspelled `aria-*` to the unique nearest valid name — same algorithm as issue 15: Levenshtein distance ≤ 2 against the shared vendored ARIA list, no unique candidate → null; bound form `:aria-*`/`v-bind:aria-*` → rename the attribute NAME only (value expression untouched) |

Shared requirements:

1. `allowedSpan` = element start-tag span within the SFC file (absolute
   offsets from vue-eslint-parser template AST).
2. Directive-bound attributes (`v-bind:x` / `:x`): only case 5's name-rename
   touches them; all other fixers return null on bound attributes.
3. Elements with `v-bind="obj"` (object spread) → all fixers null
   (`fix_skipped`, note `v-bind-object`).
4. Golden fixtures under `fixtures/fixers/vue/<rule>/` covering: static attr,
   bound attr, `v-bind` spread, multiline tags, `<script setup>` files
   (assert script block byte-identical after fix).
5. Verify loop runs the Vue profile; assert fingerprints of `<script>`-derived
   content are unaffected.

## Acceptance Criteria

- [ ] All 5 fixers registered; golden tests byte-exact; verify loop green.
- [ ] Bound/spread skip discipline tested for every fixer (value edits and
      removals skip bound forms; name-rename covers bound forms).
- [ ] `:aria-lable="x"` → `:aria-label="x"` rename covered; ambiguous typo →
      null.
- [ ] Class policy: auto_review fixers inert under default `fix.classes`.
- [ ] Security (DESIGN §14.2 T2): mutated-fixer negative tests through the
      engine validators (out-of-span, active-content newText).
- [ ] Script blocks and non-template SFC sections byte-identical post-fix.
- [ ] Idempotence double-run test.
- [ ] U5 resolution note added to DESIGN.md §2.3 (row updated: "resolved — own
      attribute-level edits, no upstream autofix") in the same PR.

## Validation

```bash
npm test -- src/fix/fixers/vue
```

## Dependencies

13, 07.

## Non-goals

`form-control-has-label` and other content_required/manual rules; template
restructuring.

## Design References

DESIGN.md §9.1–9.3, §2.3 U5, §14.2 T2; issue 07 fixability table; issue 15
(shared ARIA list + rename algorithm).
