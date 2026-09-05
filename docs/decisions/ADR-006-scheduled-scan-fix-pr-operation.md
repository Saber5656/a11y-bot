# ADR-006: Operation model — scheduled/manual full scan opens fix PRs; PR checks never write

- Status: accepted (2026-07-08, approved by product owner)

## Context

When should the bot write code? Options: dependabot-style scheduled fix PRs;
suggested-changes comments on user PRs; pushing fix commits onto user PR branches.

## Decision

- **Write path**: a scheduled (cron) or manually dispatched workflow runs
  `a11y-bot fix` over the default branch, then opens/updates a single idempotent
  fix PR (`a11y-bot/fix` branch, marker comment in body).
- **Read path**: on `pull_request`, workflows run `scan` (and optionally `audit`)
  in check mode — report + fail on new findings vs baseline. No commits, no
  comments that require write scopes in v1.

## Rationale

- Predictable and least-privilege: check jobs need `contents: read` only.
- No surprise mutations of contributor branches; no CI-trigger loops.
- Matches a proven mental model (dependabot), lowering adoption friction.

## Consequences

- Suggested-changes review comments are v2 (needs diff-position mapping + comment
  volume control).
- The fix PR is bot-owned: recreating it force-updates only `a11y-bot/*` branches,
  guarded by a branch-name allowlist and marker check (never other branches).
