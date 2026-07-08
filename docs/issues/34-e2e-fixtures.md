# Title

E2E fixture projects and full-pipeline CI tests

## Summary

Build the three fixture mini-projects and the end-to-end CI suite exercising
scan → fix → audit against them, verifying the wave-level exit criteria and
performance budgets.

## Context

DESIGN.md §16 (E2E row) and §17 budgets. Earlier issues created per-module
fixtures; this issue creates realistic mini-apps and the cross-subsystem tests
that catch integration regressions.

## Scope

- `fixtures/static-html/` (multi-page site), `fixtures/react-vite/`,
  `fixtures/vue-vite/`; `test/e2e/*.test.ts`; CI job `e2e`.

## Detailed Requirements

1. Fixture content requirements (each project):
   - ≥ 12 seeded violations spanning: missing alt (fixable via LLM stub),
     redundant role (auto_safe), positive tabindex (auto_review), missing lang
     (config-gated), invisible focus style, keyboard-unreachable control,
     < 24 px target, reflow overflow, missing title/h1, failed keyboard flow;
   - a `SEEDED_VIOLATIONS.md` manifest table in each fixture (rule → file/page
     → expected finding count) — E2E asserts against this table so fixtures
     and expectations cannot drift silently;
   - react-vite / vue-vite build with `npm run build` into `dist/` (pinned,
     minimal dependencies; no network at test time beyond the initial repo
     `npm ci` — fixture node_modules installed via workspace? NO: fixtures have
     their own package.json but tests run against prebuilt committed `dist/`
     to keep CI offline+fast; a scheduled CI job rebuilds to detect drift).
2. E2E scenarios (spawned CLI, built package):
   - scan: JSON findings match seeded manifest (count per rule);
   - baseline: adopt → clean → seed one new violation via temp copy → gate 1;
   - fix: `--dry-run` diff snapshot; apply on temp copy → re-scan → fixed
     rules gone, needs_human list matches manifest; idempotent second run;
   - fix with stub LLM server: alt text applied, marked AI-generated;
   - audit (static-html + built react/vue via `staticDir`): expected runtime
     findings per manifest; evidence bundle validates; flows incl. one failing;
   - audit with stub LLM: advisory findings present, exit unaffected;
   - `fix-pr --dry-run`: body snapshot (no network).
3. Performance assertions (soft-fail warnings in PR, hard-fail on 2× budget):
   scan of react-vite < 60 s; audit per target×viewport < 90 s (§17).
4. CI wiring: `e2e` job needs chromium (`npx playwright install --with-deps
   chromium`), runs on Node 22 only (matrix cost), uploads evidence bundle as
   artifact on failure for debugging.
5. Flake policy: E2E tests must not retry by default; any flaky test found is a
   bug to fix (determinism is a product property) — document in test README.

## Acceptance Criteria

- [ ] All scenarios green in CI from a clean checkout.
- [ ] Seeded-manifest assertion mechanism proves fixture/expectation sync
      (mutating a fixture without the manifest fails E2E — negative test).
- [ ] Performance budget checks active with recorded numbers in job summary.
- [ ] Evidence artifact-on-failure wiring verified once (forced failure in a
      draft commit, then reverted — noted in PR).
- [ ] Total `e2e` job wall-clock < 15 min.

## Validation

```bash
npm run test:e2e   # script added by this issue
```

## Dependencies

17, 27, 31.

## Non-goals

Real-provider LLM tests (manual, 33); cross-OS matrix (linux only; mac/windows
smoke is v2); visual regression testing.

## Design References

DESIGN.md §16, §17; ISSUE_PLAN wave exit criteria.
