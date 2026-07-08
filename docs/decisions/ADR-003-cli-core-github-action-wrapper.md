# ADR-003: CLI core with a thin GitHub Action wrapper (no hosted service)

- Status: accepted (2026-07-08, approved by product owner)

## Context

Delivery options considered: GitHub App (hosted, webhook-driven), GitHub Action
only, CLI only, CLI core + Action wrapper.

## Decision

Implement all functionality in an **npm-distributed CLI (`a11y-bot`)**; ship a
**JavaScript GitHub Action** in the same repository that maps Action inputs to CLI
invocations. No hosted service in v1.

## Rationale

- Same behavior locally and in CI → debuggable, testable, adoptable.
- No server, no multi-tenant secret custody, no webhook attack surface — the
  security model reduces to "runs inside the user's own CI with their token".
- npm name `a11y-bot` verified available (2026-07-08).

## Consequences

- Automation ("bot-ness") is delivered via workflow templates (scheduled fix runs,
  PR checks) rather than an installed App.
- GitHub App packaging remains a v2 candidate; the CLI core stays the single source
  of behavior either way.
- The Action bundles the CLI (committed `dist/` build) so user workflows need no
  `npm install` step.
