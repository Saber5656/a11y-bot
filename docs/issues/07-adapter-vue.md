# Title

Vue SFC adapter: vuejs-accessibility rule map

## Summary

Register the Vue profile (`vue-eslint-parser` + `eslint-plugin-vuejs-accessibility`)
and implement its exhaustive rule map to unified findings.

## Context

Same adapter pattern as issue 06. Plugin `flat/recommended` has ~21 rules
(research doc). Vue templates lint through `vue-eslint-parser`; script blocks are
not a11y-linted in v1.

## Scope

- `src/static/adapters/vue.ts`, `src/static/rule-maps/vue.ts`, registry +
  catalog entries.

## Detailed Requirements

1. Profile fragment: files `**/*.vue`, plugin `vuejs-accessibility`, rules from
   `flat/recommended`. Parser wiring (exact flat-config shape):
   ```js
   {
     languageOptions: {
       parser: vueEslintParser,
       parserOptions: {
         parser: { js: "espree", ts: tsEslintParser },  // per-lang script parsing
         ecmaVersion: "latest", sourceType: "module",
       },
     },
   }
   ```
   (`tsEslintParser` = the same `typescript-eslint` parser package pinned in
   issue 05.)
2. Rule map rows for every plugin rule using the shared `RuleMapRow` type from
   issue 05, including `enabled` and `docsUrl` per row. Normative fixability:
   | upstream rule | fixability |
   |---|---|
   | `alt-text` | content_required |
   | `no-access-key` | auto_safe |
   | `no-autofocus` | auto_review |
   | `no-redundant-roles` | auto_safe |
   | `tabindex-no-positive` | auto_review |
   | `iframe-has-title` | content_required |
   | `form-control-has-label` | content_required |
   | `no-onchange` | manual (bestPractice) |
   | `aria-props` | auto_review |
   | all remaining (`anchor-has-content`, `aria-role`,
     `aria-unsupported-elements`, `click-events-have-key-events`,
     `heading-has-content`, `interactive-supports-focus`, `label-has-for`,
     `media-has-caption`, `mouse-events-have-key-events`,
     `no-distracting-elements`, `no-static-element-interactions`,
     `role-has-required-aria-props`) | manual (v1) |

   **Normative default**: any plugin rule not named above registers as `manual`
   — the table plus this default is the exhaustive assignment.
3. WCAG refs per plugin docs; best-practice rules flagged as in issue 06.
4. Template position fidelity: findings must carry the position inside the
   `.vue` file (ESLint already reports SFC-absolute lines via vue-eslint-parser —
   add a fixture test proving line/column land inside the `<template>` block).
5. Completeness test identical in mechanism to issue 06 (iterate plugin rules).

## Acceptance Criteria

- [ ] Completeness test green; negative test included (deleting a map row makes
      it fail).
- [ ] Fixture SFCs (options API + `<script setup>` + `lang="ts"`) produce
      expected findings (≥ 6 distinct rules incl. alt-text,
      form-control-has-label, tabindex-no-positive) with correct positions.
- [ ] Registry sweep: every row has `docsUrl`, `defaultSeverity`, `wcag` (or
      bestPractice), `fixability`, `messageId`; provenance
      `source = { tool: "eslint-plugin-vuejs-accessibility", version, ruleId }`.
- [ ] `scan.rules` override (`off`/`warn`/`error`) tested for one Vue rule.
- [ ] T12 integration: a synthetic never-returning parse in a `.vue` fixture is
      terminated by the issue-05 worker timeout (shared harness test).
- [ ] Fingerprints stable across unrelated `<script>` edits (test).
- [ ] All messageIds resolve in the catalog.

## Validation

```bash
npm test -- src/static/adapters/vue src/static/rule-maps
```

## Dependencies

05.

## Non-goals

Vue fixers (16); script-block a11y analysis; Nuxt-specific resolution.

## Design References

DESIGN.md §8.1–8.3; research/2026-07-static-lint-engines.md.
