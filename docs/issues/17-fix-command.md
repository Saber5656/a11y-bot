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
     warning + content_required reported needs_human, run continues).
   - Format/output routing follows the issue-09 rules verbatim (repeatable
     `--format` replaces config; file formats → outputDir default
     `.a11ybot/reports`; console → stdout).
   - stdout contract for `--dry-run`: with console format (default) the
     unified diff IS the stdout payload (summary to stderr); with
     `--format json`, stdout carries ONLY the JSON document (diffs embedded
     per file as strings) — never both.
2. Flow: scan (same pipeline as issue 09, honoring paths narrowing) → filter
   findings with a registered fixer whose class is enabled → engine
   plan/apply/verify per file → write successful files to disk (unless
   `--dry-run`) → summary.
   Exported handoff contract (consumed by issues 29/30):
   `runFix(ctx, opts): Promise<FixRunOutput>` where
   `FixRunOutput = { results: FixResultSummary; changedFiles: Array<{ path:
   string; newContent: string }> }` — `changedFiles` carries the verified
   post-fix content so the git engine can materialize the same changes in its
   worktree without re-running fixers.
3. `--dry-run`: writes nothing; unified diff in `diff -u` format,
   repo-relative paths, stable ordering (routing per requirement 1).
4. Summary (console + `--format json|markdown` via issues 09/10 structures):
   counts of fixed / fix_skipped / fix_deferred / fix_failed / needs_human,
   grouped by ruleId; needs_human table lists content_required+manual findings
   (this is the PR body's "Needs human" source).
5. Exit codes: 0 on success (including "nothing to fix"); 2 usage/config;
   3 internal; **never** 1 (fix is not a gate). Documented in help text.
6. Safety: refuses to run if any DISCOVERED FIXABLE FILE (the candidate set
   after scan+plan filtering, not the CLI `[paths...]` args) has uncommitted
   modifications AND `--dry-run` is absent AND env `A11YBOT_ALLOW_DIRTY` unset —
   protects users from mixing bot edits with their WIP. Implementation: run
   `git status --porcelain` once, intersect with the candidate file set;
   non-empty intersection → ConfigError listing the files. Outside a git repo →
   warning, proceed.
7. Determinism: summary ordering stable; `--dry-run` twice → identical output.

## Acceptance Criteria

- [ ] Fixture repo: `fix --dry-run` shows expected diffs; `fix` applies them;
      second run reports zero plans.
- [ ] Class gating: default run applies only auto_safe;
      `--classes auto_safe,auto_review` applies both (fixture asserts counts).
- [ ] Dirty-worktree guard triggers/bypasses correctly (test in temp git repo).
- [ ] JSON summary schema documented + validated; markdown summary snapshot
      (renders via issue 10 with `fixResults`).
- [ ] `--llm` without key: warning once, content_required listed needs_human.
- [ ] Security integration (DESIGN §14.2 T2/T3/T8): with the stub LLM,
      (a) a hostile stub response is rejected by validators and the finding
      lands in needs_human, (b) summaries/warnings contain no env-canary
      values, (c) `llm.maxCalls` exhaustion mid-run degrades to needs_human
      without error exit.
- [ ] Dirty-guard precision: unrelated dirty file does NOT block; dirty
      candidate file DOES block and is named in the error.

## Validation

```bash
npm test -- src/cli/commands/fix
node dist/cli/index.js fix fixtures/static-html --dry-run
```

## Dependencies

13, 14, 15, 16, 10 (markdown summary rendering).

## Non-goals

Branch/commit/PR (29/30); LLM fixer itself (20 — integrates transparently via
catalog).

## Design References

DESIGN.md §9.4, §9.2 (results), §12.3, §14.3.
