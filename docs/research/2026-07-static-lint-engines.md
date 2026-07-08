# Research: Static a11y lint engines for reuse (2026-07)

Status: verified 2026-07-08 against npm registry and upstream docs.
Decision consumer: [ADR-002](../decisions/ADR-002-reuse-existing-lint-engines.md), DESIGN.md §8.

## Question

Which existing static analysis assets can a11y-bot reuse as detection backends for
HTML / JSX-TSX (React) / Vue SFC, instead of reimplementing dozens of a11y rules?

## Verified facts (npm registry, 2026-07-08)

| Package | Version | License | ESLint peer range | Notes |
|---|---|---|---|---|
| `eslint` | 10.6.0 (latest) | MIT | — | Flat config only since v9 |
| `eslint-plugin-jsx-a11y` | 6.10.2 | MIT | `^3 … ^9` (**no ^10**) | ~35 rules, industry standard for JSX |
| `eslint-plugin-vuejs-accessibility` | 2.5.0 | MIT | `^5 … ^10` | ~21 rules for Vue SFC templates |
| `vue-eslint-parser` | 4.12.1 (checked) | MIT | `^8.57 / ^9 / ^10` | Required to parse `.vue` |
| `@html-eslint/eslint-plugin` | 0.63.0 | MIT | `>=8.0.0 \|\| ^10.0.0-0` | Has a dedicated **Accessibility** rule category |
| `@html-eslint/parser` | (paired) | MIT | — | Parses plain `.html` for ESLint |
| `markuplint` | 4.18.3 | MIT | n/a (own engine) | Alternative HTML linter, own CLI/engine |

### @html-eslint accessibility category (verified from https://html-eslint.org/docs/rules)

17 rules: `no-abstract-roles`, `no-accesskey-attrs`, `no-aria-hidden-body`,
`no-aria-hidden-on-focusable`, `no-empty-headings`, `no-heading-inside-button`,
`no-invalid-role`, `no-non-scalable-viewport`, `no-positive-tabindex`,
`no-redundant-role` (the only upstream-fixable one), `no-skip-heading-levels`,
`require-content`, `require-form-method`, `require-frame-title`, `require-img-alt`,
`require-input-label`, `require-meta-viewport`.

Implication: upstream autofix coverage is nearly zero — a11y-bot's own fix engine is
the value-add, not a duplication.

## Key constraint discovered

`eslint-plugin-jsx-a11y@6.10.2` does **not** declare ESLint 10 peer support while the
other plugins do. Running everything on ESLint 10 would require `--legacy-peer-deps`
or overrides (fragile for an OSS install base).

**Resolution:** pin the embedded engine to the latest ESLint 9.x. The engine is an
internal dependency of a11y-bot (installed inside our package, isolated flat config,
never reads the user's ESLint setup), so the pin is invisible to users. Revisit when
jsx-a11y publishes ESLint 10 support (tracked as a known unknown).

## Options considered for HTML

| Option | Pros | Cons |
|---|---|---|
| `@html-eslint` (chosen) | Same ESLint pipeline as JSX/Vue → one engine, one finding mapper; MIT; active; dedicated a11y category | Fewer a11y rules than markuplint presets |
| `markuplint` | Rich a11y presets, ARIA spec-aware | Second engine to embed (different config, results, versioning); heavier integration surface |
| Custom parse5 rules | Full control | Reimplementation + false-positive tuning cost, delays v1 |

Chosen: `@html-eslint` for v1 (single-engine architecture). markuplint re-evaluation
is a v2 candidate if HTML rule depth becomes a limiter.

## Rule count summary (recommended presets)

| Source | Approx. rules | Upstream fixable |
|---|---|---|
| jsx-a11y recommended | ~35 | ~2 |
| vuejs-accessibility recommended | ~21 | ~2 |
| @html-eslint a11y category | 17 | 1 |

a11y-bot maps each upstream rule id to a **unified rule registry** entry
(`static/<source>/<rule>`) with WCAG references, severity, and fixability class.
Completeness is enforced by a test that iterates the pinned plugin's exported rules
and fails on unmapped entries (see issue 06/07/08).
