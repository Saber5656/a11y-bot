# ADR-002: Reuse existing ESLint a11y plugins as the static detection backend

- Status: accepted (2026-07-08, approved by product owner)

## Context

Static detection for HTML/JSX/Vue needs dozens of battle-tested rules. Options:
reimplement rules on our own AST layer, or embed mature engines. Research:
[2026-07-static-lint-engines](../research/2026-07-static-lint-engines.md).

## Decision

Embed **ESLint (pinned to 9.x)** programmatically as an internal engine with an
isolated flat config (never reading the user's ESLint setup), using:

- `eslint-plugin-jsx-a11y` for `.jsx/.tsx`
- `eslint-plugin-vuejs-accessibility` + `vue-eslint-parser` for `.vue`
- `@html-eslint/eslint-plugin` + `@html-eslint/parser` (Accessibility category) for `.html`

Upstream rule ids are mapped into a11y-bot's unified rule registry
(`static/<source>/<rule>`) carrying WCAG refs, normalized severity, and fixability
class. a11y-bot's differentiation lives in the fix engine, PR automation, runtime
audit, and reporting — not in rule reimplementation.

ESLint is pinned to 9.x because `eslint-plugin-jsx-a11y@6.10.2` lacks ESLint 10 peer
support (verified 2026-07-08). Upgrade is a tracked known unknown.

## Consequences

- v1 reaches useful detection quality quickly with low false-positive risk.
- Rule granularity/ids follow upstream; the registry mapping layer absorbs this
  (stable public rule ids even if upstream renames).
- A custom-rule extension point is NOT part of v1 (kept as v2) to protect scope.
- Version bumps of plugins require re-running the mapping completeness test.
