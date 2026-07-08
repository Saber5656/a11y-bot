# Title

HTML adapter: @html-eslint a11y rule map

## Summary

Register the plain-HTML profile (`@html-eslint/parser` + plugin) enabling the
Accessibility category plus selected a11y-relevant extras, with its exhaustive
rule map.

## Context

Research doc verified the Accessibility category (17 rules, only
`no-redundant-role` upstream-fixable). DESIGN.md §8.1 adds extras from other
categories: `require-lang` (WCAG 3.1.1) and `require-title` (2.4.2); this issue
fixes the exact extra set.

## Scope

- `src/static/adapters/html.ts`, `src/static/rule-maps/html.ts`, registry +
  catalog entries.

## Detailed Requirements

1. Profile fragment: files `**/*.html`, parser `@html-eslint/parser`, plugin
   `@html-eslint`.
2. Enabled rules = the 17 Accessibility-category rules — `no-abstract-roles`,
   `no-accesskey-attrs`, `no-aria-hidden-body`, `no-aria-hidden-on-focusable`,
   `no-empty-headings`, `no-heading-inside-button`, `no-invalid-role`,
   `no-non-scalable-viewport`, `no-positive-tabindex`, `no-redundant-role`,
   `no-skip-heading-levels`, `require-content`, `require-form-method`,
   `require-frame-title`, `require-img-alt`, `require-input-label`,
   `require-meta-viewport` — plus extras `require-lang`, `require-title`
   (verify exact rule ids against the pinned plugin version; if an extra does
   not exist under that id, document the actual id in the rule map and update
   DESIGN.md §8.1 in the same PR).
3. Rule map rows for every **enabled** rule, using the shared `RuleMapRow` type
   from issue 05 (incl. `docsUrl` per row). Scope note: per DESIGN §8.2, this
   adapter's exhaustiveness is defined over the enabled a11y set — the
   completeness test iterates the enabled set and additionally asserts each
   enabled rule id exists in the installed plugin (typo guard). The plugin's
   style/SEO categories stay out of scope. Normative fixability:
   | upstream rule | fixability |
   |---|---|
   | `no-redundant-role` | auto_safe |
   | `no-aria-hidden-body` | auto_safe |
   | `no-accesskey-attrs` | auto_safe |
   | `no-positive-tabindex` | auto_review |
   | `no-abstract-roles`, `no-invalid-role` | auto_review (remove role attr) |
   | `require-lang` | auto_safe with `configGated: "fix.defaults.lang"` (DESIGN §7.2) |
   | `require-img-alt` | content_required |
   | `require-frame-title` | content_required |
   | `require-input-label` | content_required |
   | `no-non-scalable-viewport`, `require-meta-viewport` | auto_review |
   | `no-aria-hidden-on-focusable` | auto_review (remove aria-hidden) |
   | `no-empty-headings`, `no-heading-inside-button`, `no-skip-heading-levels`, `require-content`, `require-form-method`, `require-title` | manual (v1) |
4. WCAG refs per rule from plugin docs / WCAG mapping (e.g., require-img-alt →
   1.1.1 A; no-positive-tabindex → 2.4.3 A; no-non-scalable-viewport → 1.4.4 AA;
   require-title → 2.4.2 A). bestPractice flag where no SC applies.
5. Conversion identical to issues 06/07 via `createFinding`.

## Acceptance Criteria

- [ ] Completeness test over the enabled rule set green; negative tests: a map
      row for a nonexistent plugin rule id fails, and deleting a row for an
      enabled rule fails.
- [ ] Fixture HTML pages produce expected findings (≥ 8 rules incl.
      require-img-alt, no-positive-tabindex, no-invalid-role, require-lang).
- [ ] Extra-rule ids verified against the pinned plugin (test imports the rule
      objects directly — a typo fails at test time, not runtime).
- [ ] Registry sweep: every row has `docsUrl` and full metadata; provenance
      `source = { tool: "@html-eslint/eslint-plugin", version, ruleId }`.
- [ ] `scan.rules` override tested for one HTML rule.
- [ ] T12 integration: hostile fixture (1 MiB+ single-line HTML) is skipped by
      discovery; a synthetic parser hang is terminated by the issue-05 worker
      timeout (shared harness test).
- [ ] All messageIds resolve in the catalog.

## Validation

```bash
npm test -- src/static/adapters/html
```

## Dependencies

05.

## Non-goals

HTML fixers (14); style/SEO categories; markuplint evaluation (v2).

## Design References

DESIGN.md §7.1–7.4 (finding/registry/severity), §8.1–8.3, §14.2 T12;
research/2026-07-static-lint-engines.md.
