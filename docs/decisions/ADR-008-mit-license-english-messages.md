# ADR-008: MIT license; English-only user-facing messages in v1 (i18n-ready catalog)

- Status: accepted (2026-07-08, approved by product owner)

## Decision

1. **License: MIT.** Compatible with every embedded dependency (MIT plugins,
   Apache-2.0 Playwright, MPL-2.0 axe-core consumed unmodified as a dependency).
2. **All user-facing output (findings, reports, PR bodies, CLI help) is English**
   in v1 — maximizes OSS reach; owner confirmed 2026-07-08.
3. Messages live in a **catalog keyed by unified rule id / message id**, not inline
   strings, so a `ja` catalog can be added in v2 without touching engine code.
   JIS X 8341-3 correspondence is documented in English docs (see research:
   [2026-07-standards-wcag-jis](../research/2026-07-standards-wcag-jis.md)).

## Consequences

- No locale plumbing in v1 beyond the catalog indirection (cheap now, enabling later).
- GitHub-facing artifacts (Issues/PRs the bot creates) are English, matching the
  repository language policy.
