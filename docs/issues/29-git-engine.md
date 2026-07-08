# Title

Git engine: bot branch, rule-grouped commits, allowlist guard

## Summary

Implement the git layer that turns applied fixes into commits on a bot-owned
branch, with hard safety guards on which refs may ever be touched.

## Context

DESIGN.md §13.1 is normative. This engine is invoked by the PR pipeline
(scheduled workflow → `fix` → THIS → PR engine 30). Safety property T6: the
engine must be physically unable to modify non-`a11y-bot/*` refs.

## Scope

- `src/github/git.ts`, `src/github/identity.ts`.

## Detailed Requirements

1. Implementation via child-process `git` (no libgit dependency); every
   invocation logged at debug with args (never env); repo root detected via
   `git rev-parse --show-toplevel` (not a repo → EnvError exit 4).
2. Preconditions API `assertCleanBase()`: working tree clean
   (`status --porcelain` empty) and HEAD detached-or-branch recorded; violations
   → ConfigError with hint (CI checkout guidance).
3. Branch lifecycle:
   - `prepareBotBranch(name)`: full ref = `refs/heads/<github.branchPrefix>fix`
     (default `a11y-bot/fix`); **guard**: computed branch name MUST match
     `^a11y-bot\/[a-z0-9._-]+$` after prefix substitution — otherwise
     InternalError (guard is a pure function, unit-tested exhaustively;
     config prefix override that violates the pattern → ConfigError at load —
     add the check to issue 02's schema via refinement in THIS pr);
   - branch created from current HEAD; if it exists locally, hard-reset it to
     HEAD (it is bot-owned by definition of the guard);
   - `checkout` into a temporary worktree (`git worktree add`) so the user's
     checkout is never disturbed; fixes are re-applied there by replaying the
     recorded FixResult file edits (engine receives the edit log, not dirty
     files) — teardown removes the worktree.
4. Commits: group FixResults by unified ruleId (stable order); one commit per
   group: subject `fix(a11y): <ruleId> — <n> file(s)`, `[llm]` suffix when any
   plan in group is llmGenerated; body lists files + finding fingerprints;
   author/committer from `identity.ts`: `a11y-bot <a11y-bot[bot]@users.noreply.github.com>`
   (override via `github.commitIdentity` — ADD to config schema §6.2 in this
   PR); `--no-gpg-sign`, `--no-verify` NOT used (user hooks run; a hook
   rejection surfaces as EnvError with the hook output).
5. Push: `pushBotBranch()` uses transient authenticated URL
   `https://x-access-token:<token>@github.com/<owner>/<repo>.git` passed as an
   inline remote argument (never `remote add`, never credential store);
   `--force-with-lease` allowed ONLY because the guard restricts to
   `a11y-bot/*` refs (assert again immediately before push); token from
   `GITHUB_TOKEN`/`GH_TOKEN` (absent → EnvError with hint).
   Owner/repo resolved from `origin` remote URL (https or ssh forms parsed;
   no origin → EnvError).
6. Dry-run mode (`--dry-run` plumbed from the caller): everything except push;
   prints would-push summary.

## Acceptance Criteria

- [ ] In a temp repo: end-to-end prepare→commit→(mock)push produces expected
      branch/commits; user worktree untouched (status clean).
- [ ] Guard tests: prefix overrides like `main`, `a11y-bot/../main`, empty →
      rejected at the right layer (schema or InternalError).
- [ ] Commit grouping/message format snapshot; `[llm]` marker logic tested.
- [ ] Token never in logs (redaction integration test); push URL never persisted
      (`git remote -v` clean, `git config --list` clean).
- [ ] Failing pre-commit hook fixture → EnvError with hook output.
- [ ] ssh and https origin parsing tested.

## Validation

```bash
npm test -- src/github/git
```

## Dependencies

17.

## Non-goals

PR creation (30); GitHub API use (30); rebasing existing bot branches onto
moved bases (bot branch is always recreated from HEAD).

## Design References

DESIGN.md §13.1, §14.2 T3/T6; ADR-006.
