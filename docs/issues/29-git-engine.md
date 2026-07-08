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
   invocation logged at debug with args passed through the issue-04 redaction
   filter (and by construction args never contain tokens — see requirement 5);
   repo root detected via `git rev-parse --show-toplevel` (not a repo →
   EnvError exit 4).
2. Preconditions API `assertCleanBase()`: working tree clean
   (`status --porcelain` empty) and HEAD detached-or-branch recorded; violations
   → ConfigError with hint (CI checkout guidance).
3. Branch lifecycle:
   - `prepareBotBranch(name)`: full ref = `refs/heads/<github.branchPrefix>fix`
     (default `a11y-bot/fix`); **guard**: computed branch name MUST match
     `^a11y-bot\/[a-z0-9._\/-]+$` (same charset as issue 02's schema pattern)
     — otherwise InternalError (defense in depth; the schema already prevents
     this at load. Guard is a pure function, unit-tested exhaustively);
   - branch created from current HEAD; if it exists locally, hard-reset it to
     HEAD (it is bot-owned by definition of the guard);
   - `checkout` into a temporary worktree (`git worktree add`) so the user's
     checkout is never disturbed; changes materialize by writing the
     `FixRunOutput.changedFiles` contents (issue 17's exported contract) into
     the worktree — teardown removes the worktree.
4. Commits: group changed files by unified ruleId from
   `FixRunOutput.results` (stable order); one commit per group with the exact
   DESIGN §13.1 subject template `fix(a11y): <unified ruleId> (<N> files)`,
   suffix ` [llm]` (constant from issue 19) when any entry in the group is
   llmGenerated; body lists files + finding fingerprints; author/committer
   from `github.commitIdentity` (schema key from issue 02; `identity.ts` just
   resolves it); `--no-gpg-sign`, `--no-verify` NOT used (user hooks run; a
   hook rejection surfaces as EnvError with the hook output).
5. Push: `pushBotBranch()` authenticates via environment-injected git config
   (DESIGN §13.2): `GIT_CONFIG_COUNT=1`,
   `GIT_CONFIG_KEY_0=http.https://github.com/.extraheader`,
   `GIT_CONFIG_VALUE_0=AUTHORIZATION: basic <base64(x-access-token:TOKEN)>` —
   the token never appears in argv, `ps`, logs, or persisted config. Push URL
   is the plain `https://github.com/<owner>/<repo>.git`. `--force-with-lease`
   allowed ONLY for guard-matching refs (assert again immediately before
   push); token from `GITHUB_TOKEN`/`GH_TOKEN` (absent → EnvError with hint).
   Owner/repo resolved from `origin` remote URL (https or ssh forms parsed;
   no origin → EnvError).

## Acceptance Criteria

- [ ] In a temp repo: end-to-end prepare→commit→(mock)push produces expected
      branch/commits; user worktree untouched (status clean).
- [ ] Guard tests: prefix overrides like `main`, `a11y-bot/../main`, empty →
      rejected at the right layer (schema or InternalError); the pre-push
      allowlist assertion fires on a synthetic non-matching refspec.
- [ ] Default branch provably untouched (T6): after a full run in the temp
      repo, `git rev-parse main` unchanged and reflog shows no writes to any
      non-`a11y-bot/*` branch ref.
- [ ] Commit grouping/message format snapshot matches the §13.1 template;
      ` [llm]` marker logic tested.
- [ ] Token security (T3): token never in argv (spawn-call inspection test),
      never in logs (redaction integration test), never persisted
      (`git remote -v` clean, `git config --list` clean after run).
- [ ] Failing pre-commit hook fixture → EnvError with hook output.
- [ ] ssh and https origin parsing tested.

## Validation

```bash
npm test -- src/github/git
```

## Dependencies

17 (`FixRunOutput` contract).

## Non-goals

PR creation (30); GitHub API use (30); dry-run handling (the `fix-pr` command
stops before this engine on `--dry-run` — issue 30); rebasing existing bot
branches onto moved bases (bot branch is always recreated from HEAD).

## Design References

DESIGN.md §13.1, §14.2 T3/T6; ADR-006.
