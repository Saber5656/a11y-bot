# Title

JSX/TSX adapter: jsx-a11y rule map to unified findings

## Summary

Register the `eslint-plugin-jsx-a11y` profile and implement the exhaustive rule
map converting its results into unified findings.

## Context

DESIGN.md §8.2: adapters own exhaustive per-rule tables (unified id, WCAG refs,
severity, fixability) validated by a completeness test that iterates the pinned
plugin's exported rules. jsx-a11y `flat/recommended` has ~35 rules (research doc).

## Scope

- `src/static/adapters/jsx.ts` (profile fragment + result conversion).
- `src/static/rule-maps/jsx.ts` (the table).
- Registry entries + message catalog entries for every mapped rule.

## Detailed Requirements

1. Profile fragment: files `**/*.{jsx,tsx}`, `typescript-eslint` parser
   (jsx enabled, no type info), plugin `jsx-a11y`, rules = every rule in
   `flat/recommended` at its recommended level (off-by-default upstream rules
   like `control-has-associated-label` are included in the table with
   `defaultSeverity` noted and rule turned off — table row exists regardless).
2. Rule map: one row per plugin rule. Row shape (compile-time typed):
   ```ts
   { upstream: "alt-text", unified: "static/jsx-a11y/alt-text",
     wcag: [{ sc: "1.1.1", level: "A", version: "2.0" }],
     defaultSeverity: "serious", fixability: "content_required",
     messageId: "jsx-a11y.alt-text" }
   ```
   Normative fixability assignments (the fix engine keys off these):
   | upstream rule | fixability |
   |---|---|
   | `alt-text` | content_required |
   | `img-redundant-alt` | auto_review |
   | `aria-props` | auto_review |
   | `no-access-key` | auto_safe |
   | `no-autofocus` | auto_review |
   | `no-redundant-roles` | auto_safe |
   | `tabindex-no-positive` | auto_review |
   | `anchor-is-valid`, `click-events-have-key-events`, `label-has-associated-control`, `media-has-caption`, all `no-noninteractive-*`, `no-static-element-interactions`, `interactive-supports-focus`, `role-has-required-aria-props`, `role-supports-aria-props`, `aria-role`, `aria-proptypes`, `aria-unsupported-elements`, `aria-activedescendant-has-tabindex`, `autocomplete-valid`, `heading-has-content`, `anchor-has-content`, `html-has-lang`*, `lang`*, `iframe-has-title`*, `mouse-events-have-key-events`, `no-distracting-elements`, `scope`, remaining rules | manual (v1) |

   *`html-has-lang`/`lang`: fixability `auto_safe` only when `fix.defaults.lang`
   is configured (engine-level condition; table stores `auto_safe_config_gated`,
   a fixability modifier defined in issue 03 as metadata flag
   `configGated: "fix.defaults.lang"`). `iframe-has-title`: content_required.
3. WCAG refs: assign per rule from the jsx-a11y docs' WCAG annotations; rules
   documented as best-practice-only get `bestPractice: true` and empty wcag.
   The implementer records the mapping source URL per row as a code comment.
4. Conversion: `LintResult` → findings via issue 03 `createFinding`
   (file, 1-based range from ESLint message, snippet from source line).
5. **Completeness test** (normative): iterate
   `Object.keys(jsxA11y.rules)` of the installed plugin; every rule must have a
   table row, and every table row must exist in the plugin. Recommended-set
   drift (rule added/removed upstream) fails this test with a clear message.

## Acceptance Criteria

- [ ] Completeness test green against pinned plugin version; deliberately
      removing a row makes it fail (negative test included).
- [ ] Fixture React components produce expected findings (≥ 8 distinct rules
      covered, incl. alt-text, aria-props typo, tabindex positive, no-access-key)
      with correct unified ids, wcag arrays, fingerprints (snapshot test).
- [ ] `scan.rules` severity overrides (`off`/`warn`/`error`) applied via the
      shared mechanism (§8.3) — at least one override tested here.
- [ ] All mapped messageIds resolve in the English catalog (no fallback logs).

## Validation

```bash
npm test -- src/static/adapters/jsx src/static/rule-maps
```

## Dependencies

05.

## Non-goals

Fix implementations (15); Vue/HTML rules (07/08).

## Design References

DESIGN.md §7.2, §8.2–8.3, §9.1; research/2026-07-static-lint-engines.md.
