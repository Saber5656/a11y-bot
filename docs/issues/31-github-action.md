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
   - `runs.using`: the newest Node runtime GitHub Actions supports at
     implementation time (check docs; `node24` expected — verify, record in
     code comment); `main: dist/action/index.js`.
   - Inputs (all strings per Actions model): `mode` (required: `check|fix|audit`),
     `config-path`, `fail-on`, `llm` (`"true"|"false"`, default false),
     `github-token` (default `${{ github.token }}`), `update-baseline`
     (default false), `paths` (space-separated, optional).
   - Outputs: `summary-json-path`, `new-findings-count`, `pr-url` (fix mode).
2. Entry (`action/index.ts`, bundled with the same tsup/ncc-style single-file
   build into `dist/action/`):
   - maps inputs → CLI argv: `check`→`scan --format json --format markdown
     --output <runner temp>`, `fix`→`fix-pr`, `audit`→`audit …`;
   - sets `GITHUB_TOKEN` env for child from `github-token` input (masked by
     Actions natively);
   - invokes the CLI **in-process** (import the command entry, not a child
     shell) to keep one bundle and correct exit propagation; exit code → action
     failure with annotation;
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
4. Bundle hygiene: `dist/action/index.js` committed by CI check (staleness:
   `npm run build && git diff --exit-code dist/` job) — decided over
   release-only builds so `uses: owner/a11y-bot@main` works for early adopters;
   license headers of bundled deps preserved (bundler license comment option).
5. Smoke workflow (this repo): on PR, runs the action from the local checkout
   (`uses: ./`) in `mode: check` against `fixtures/static-html` expecting
   exit failure→pass logic: run with `fail-on: critical` on a fixture with no
   critical findings → success; assert step summary file non-empty.

## Acceptance Criteria

- [ ] `uses: ./` smoke workflow green in CI (check mode), summary rendered.
- [ ] Input mapping unit tests (each mode, defaults, bad mode → failure
      annotation).
- [ ] dist staleness CI check active; bundle runs on a clean runner image
      (no node_modules) — verified by the smoke job.
- [ ] Outputs (`new-findings-count`, `summary-json-path`) asserted in smoke via
      a follow-up step.
- [ ] Runtime (`node24` vs `node20`) verified against current Actions docs and
      recorded.

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

DESIGN.md §13.3, §17; ADR-003.
