# Title

PR engine: idempotent create/update, body, labels, token handling

## Summary

Implement pull-request creation/update for the bot branch: discovery by marker,
body assembly from the markdown report, labels, and the `fix-pr` CLI entry that
chains fix → git → PR.

## Context

DESIGN.md §13.2. Operation model per ADR-006: one idempotent PR per repo,
updated on subsequent runs. This issue also adds the composite CLI command the
scheduled workflow calls.

## Scope

- `src/github/pr.ts` (octokit REST), CLI `a11y-bot fix-pr` in
  `src/cli/commands/fix-pr.ts`.

## Detailed Requirements

1. Dependency: `@octokit/rest` (REST only, no GraphQL). Auth token resolution
   shared with 29; `baseUrl` override honoring `GITHUB_API_URL` env (GHES
   compatibility, best-effort).
2. Discovery: `GET /repos/{o}/{r}/pulls?head={owner}:{branch}&state=open`;
   among results, the PR whose body contains marker `<!-- a11y-bot:fix-pr -->`
   is "ours". Open PR on the branch WITHOUT marker → abort with EnvError
   (never hijack a human's PR; hint to close/rename).
3. Create path: base = repo default branch (from `GET /repos`), head = bot
   branch, title from `github.prTitle`, body = marker + markdown report (10,
   `kind: "fix"`) + reproduction footer; `maintainer_can_modify: true`.
4. Update path: PATCH title/body; body regenerated (not appended). Comment
   history is not used in v1 (no per-run comments — body edit only, avoids
   notification spam; document).
5. Labels: ensure each `github.labels` exists (`GET /labels`, create missing
   with neutral color `ededed` unless `--no-create-labels`), then set on the
   PR (issue labels endpoint). Label create failure (403 fine-grained token) →
   warning, continue (labels are best-effort).
6. Zero-fix runs: if the fix stage produced no commits, do NOT open a PR; if a
   bot PR exists and the branch now has no diff vs base, close it with a
   comment "no remaining automated fixes" (config
   `github.closeEmptyPr: true` default — ADD to schema §6.2 in this PR).
7. `fix-pr` command: `a11y-bot fix-pr [--llm] [--classes …] [--dry-run]`
   pipeline: assertCleanBase → scan+fix (17, in-memory edit log) → 29
   (branch/commits/push) → this PR step → prints PR URL (also to
   `GITHUB_OUTPUT` as `pr-url` when the env var exists). `--dry-run` stops
   after printing would-be body to stdout.
8. Failure handling: API 403/404 → EnvError with permission hint table
   (`contents: write`, `pull-requests: write`); secondary rate limit (403 +
   retry-after) → single retry after wait; still failing → EnvError.
9. Security: token only via env; body content is markdown-escaped by 10;
   marker uniqueness enforced (one marker instance; strip accidental
   duplicates from regenerated bodies).

## Acceptance Criteria

- [ ] Mocked-octokit tests: create path, update path, foreign-PR abort,
      label ensure/set, empty-run close path, permission-error hints.
- [ ] `fix-pr --dry-run` on fixture prints body without network (integration).
- [ ] Idempotence: two consecutive runs with identical findings → second run
      results in PATCH (no duplicate PR) — mocked sequence test.
- [ ] `GITHUB_OUTPUT` wiring tested via temp file.
- [ ] Manual validation against a scratch GitHub repository executed and
      transcript linked in the PR that closes this issue (release-gate
      requirement from ISSUE_PLAN §Validation.6).

## Validation

```bash
npm test -- src/github/pr src/cli/commands/fix-pr
```

## Dependencies

29, 10.

## Non-goals

Review-comment posting; assignees/reviewers config (v2); GraphQL; GitHub App
auth.

## Design References

DESIGN.md §13.2, §14.2 T3/T4; ADR-003, ADR-006.
