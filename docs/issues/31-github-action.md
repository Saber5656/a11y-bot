# Title

GitHub Action wrapper: action.yml, bundled dist, step summary

## Summary

Ship the repository as a usable GitHub Action: manifest, bundled runtime,
input→CLI mapping, step-summary output, and an in-repo smoke workflow.

## Context

ADR-003: the Action is a thin wrapper over the CLI; behavior lives in the CLI.
DESIGN.md §13.3 fixes inputs/outputs. The Action must work without
`npm install` in user workflows (bundled dist committed/released).

## Scope

- `action.yml`, `action/index.ts` (entry), bundling config, smoke workflow
  `.github/workflows/action-smoke.yml`.

## Detailed Requirements

1. `action.yml`:
   - `runs.using: node24` (required per DESIGN §13.3 — the bundle targets
     Node ≥ 22, Node 20 is EOL; verify node24 remains the current Actions
     runtime and record the docs URL in a comment. If node24 were ever
     unavailable, STOP and escalate — no silent fallback);
     `main: dist/action/index.js`.
   - Inputs (all strings per Actions model; exactly the DESIGN §13.3 set):
     `mode` (required: `check|fix|audit`), `config-path`, `fail-on`,
     `llm` (`"true"|"false"`, default false), `github-token` (default
     `${{ github.token }}`), `update-baseline` (default false).
   - Outputs: `summary-json-path`, `new-findings-count`, `pr-url` (fix mode).
2. Entry (`action/index.ts`, bundled with the same tsup/ncc-style single-file
   build into `dist/action/`):
   - normative input→argv mapping table:
     | mode | argv | applicable extra inputs |
     |---|---|---|
     | check | `scan --format json --format markdown --output <RUNNER_TEMP>/a11ybot` + (`--config <config-path>` if set) + (`--fail-on <fail-on>` if set) + (`--update-baseline` if `"true"`) | all except llm (warn if `llm` set: no LLM in check mode) |
     | fix | `fix-pr` + (`--config …`) + (`--llm` if `llm=="true"`) | fail-on ignored with warning; update-baseline ignored with warning |
     | audit | `audit --format json --format markdown --output <RUNNER_TEMP>/a11ybot` + (`--config …`) + (`--fail-on …`) + (`--llm` if `"true"`) + (`--update-baseline` if `"true"`) | — |
   - token handling: `core.setSecret(githubToken)` FIRST, then set
     `process.env.GITHUB_TOKEN = githubToken` before invoking the in-process
     command entry (no child shell, no shell interpolation of any input);
     exit code → action failure with annotation;
   - writes the markdown report to `GITHUB_STEP_SUMMARY` (truncate 1 MiB per
     Actions limit);
   - emits outputs via `GITHUB_OUTPUT`;
   - `llm: "true"` without a key in env → warning annotation, continue
     (mirrors CLI semantics).
3. Playwright note: audit mode requires browsers — the entry detects missing
   chromium and fails with an annotation instructing
   `npx playwright install --with-deps chromium` as a prior step (templates in
   32 include it). The action itself never installs browsers (keeps check/fix
   fast).
4. Package metadata: add `"action.yml"` to package.json `files` (issue 01 left
   it out because the file did not exist yet); `npm pack --dry-run` allowlist
   check updated accordingly.
5. Bundle hygiene: `dist/action/index.js` committed by CI check (staleness:
   `npm run build && git diff --exit-code dist/` job) — decided over
   release-only builds so `uses: owner/a11y-bot@main` works for early adopters;
   license headers of bundled deps preserved (bundler license comment option).
6. Smoke workflow (this repo): on PR, runs the action from the local checkout
   (`uses: ./`) in `mode: check` against `fixtures/static-html` expecting
   exit failure→pass logic: run with `fail-on: critical` on a fixture with no
   critical findings → success; assert step summary file non-empty.

## Acceptance Criteria

- [ ] `uses: ./` smoke workflow green in CI (check mode), summary rendered.
- [ ] Input mapping unit tests (each mode per the mapping table, defaults,
      bad mode → failure annotation, ignored-input warnings).
- [ ] dist staleness CI check active; bundle runs on a clean runner image
      (no node_modules) — verified by the smoke job.
- [ ] Outputs (`new-findings-count`, `summary-json-path`) asserted in smoke via
      a follow-up step.
- [ ] Security (DESIGN §14.2 T3/T4): `core.setSecret` called before any use
      (unit test on entry); no `exec`/shell invocation of input values (grep
      test: no `child_process` in action entry); check-mode smoke asserts the
      workspace is unmodified after the run (`git status --porcelain` empty)
      and that the action itself uploaded no artifacts.
- [ ] `runs.using: node24` verified against current Actions docs; source URL
      recorded in action.yml comment.

## Validation

```bash
npm run build && git diff --exit-code dist/
# CI: .github/workflows/action-smoke.yml
```

## Dependencies

09, 17, 27, 30.

## Non-goals

Marketplace publishing metadata polish (36); composite/docker action forms;
browser installation inside the action.

## Design References

DESIGN.md §13.3, §14.2 T3/T4, §17; ADR-003, ADR-007 (runtime floor).
