# Title

`fix` command: local apply, dry-run diff, summary

## Summary

Wire the fix engine and fixer batches into `a11y-bot fix` with `--dry-run`
unified diffs, class selection, and the fix summary output.

## Context

DESIGN.md §9.4. This command is also the core the scheduled workflow calls
before the git/PR engines (29/30) take over the produced worktree changes.

## Scope

- `src/cli/commands/fix.ts` (replace stub).

## Detailed Requirements

1. CLI: `a11y-bot fix [paths...] [--dry-run] [--llm]
   [--classes auto_safe[,auto_review]] [--format console|json|markdown]
   [--output <dir>]`.
   - `--classes` overrides `fix.classes`; values outside the enum → exit 2.
   - `--llm` sets `fix.llm: true` for the run (still requires key; without key →
     warning + content_required skipped, run continues).
2. Flow: scan (same pipeline as issue 09, honoring paths narrowing) → filter
   findings with a registered fixer whose class is enabled → engine
   plan/apply/verify per file → write successful files to disk (unless
   `--dry-run`) → summary.
3. `--dry-run`: writes nothing; prints a unified diff (`diff -u` format,
   repo-relative paths, stable ordering) to stdout; JSON format embeds the diff
   string per file.
4. Summary (console + `--format json|markdown` via issues 09/10 structures):
   counts of fixed / fix_skipped / fix_deferred / fix_failed / needs_human,
   grouped by ruleId; needs_human table lists content_required+manual findings
   (this is the PR body's "Needs human" source).
5. Exit codes: 0 on success (including "nothing to fix"); 2 usage/config;
   3 internal; **never** 1 (fix is not a gate). Documented in help text.
6. Safety: refuses to run if the target files have uncommitted modifications
   AND `--dry-run` is absent AND env `A11YBOT_ALLOW_DIRTY` unset — protects
   users from mixing bot edits with their WIP (git check via `git status
   --porcelain -- <paths>`; outside a git repo → warning, proceed).
7. Determinism: summary ordering stable; `--dry-run` twice → identical output.

## Acceptance Criteria

- [ ] Fixture repo: `fix --dry-run` shows expected diffs; `fix` applies them;
      second run reports zero plans.
- [ ] Class gating: default run applies only auto_safe;
      `--classes auto_safe,auto_review` applies both (fixture asserts counts).
- [ ] Dirty-worktree guard triggers/bypasses correctly (test in temp git repo).
- [ ] JSON summary schema documented + validated; markdown summary snapshot.
- [ ] `--llm` without key: warning once, content_required listed needs_human.

## Validation

```bash
npm test -- src/cli/commands/fix
node dist/cli/index.js fix fixtures/static-html --dry-run
```

## Dependencies

13, 14, 15, 16.

## Non-goals

Branch/commit/PR (29/30); LLM fixer itself (20 — integrates transparently via
catalog).

## Design References

DESIGN.md §9.4, §9.2 (results), §12.3, §14.3.
