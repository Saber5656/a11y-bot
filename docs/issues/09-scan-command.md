# Title

`scan` command: orchestration, console/JSON output, gating

## Summary

Wire discovery → adapters → findings into the `a11y-bot scan` command with
console and JSON reporters, severity overrides, and the CI gate (exit 1) logic.

## Context

First user-visible vertical slice. Baseline subtraction arrives in issue 12; this
issue implements gating as "all findings ≥ failOn" with a clean seam for the
baseline filter. DESIGN.md §12.1/§12.3 normative.

## Scope

- `src/cli/commands/scan.ts` (replace stub), `src/report/console.ts`,
  `src/report/json.ts`, shared `applyRuleControls()` (§8.3) used by all adapters.

## Detailed Requirements

1. CLI: `a11y-bot scan [paths...] [--format console|json|markdown|sarif]…
   [--output <dir>] [--fail-on critical|serious|moderate|minor] [--strict]
   [--update-baseline]` — markdown/sarif/baseline flags parse now but error
   "not implemented (issue 10/11/12)" until those land (exit 3), so flag surface
   is stable from day one.
2. Orchestration: load config → discover (05) → run profiles (06–08 registered)
   → collect findings → `applyRuleControls` (scan.rules overrides: `off` drops,
   `warn` forces non-gating, `error` marks gating at registry severity) → sort
   findings (file, line, ruleId) → report → gate.
3. Console reporter: per-file sections `<path>:<line>:<col>  <severity>
   <ruleId>  <message>`, severity-colored, summary block (counts by severity,
   files scanned, duration). `--quiet` prints summary only.
4. JSON reporter: `{ schemaVersion: 1, run: { runId, tool: {name, version},
   startedAt, durationMs, configHash }, findings: Finding[], summary: { total,
   bySeverity, byRuleId, gatingCount } }`; to stdout by default, to
   `<outputDir>/scan.json` when `--output`/config set. Must validate against
   `schemas/finding.schema.json` container schema (extend schema gen).
5. Gate: exit 1 iff any finding has `severity ≥ failOn` AND not `advisory` AND
   not downgraded to `warn` (baseline filter slot returns all findings for now).
   `failOn` default from config (`serious`).
6. Performance: parallelize profiles with `Promise.all`; budget test fixture
   (200 files) must complete < 10 s in CI (soft assertion logged, not failing).

## Acceptance Criteria

- [ ] On fixture repo with known violations: correct findings count, ordering,
      and exit 1; `--fail-on critical` with no critical findings → exit 0.
- [ ] `scan.rules` `off|warn|error` behaviors verified end-to-end.
- [ ] JSON output validates against schema; stdout stays machine-clean
      (diagnostics on stderr) — piped test.
- [ ] `[paths...]` narrowing works and non-existent path → ConfigError exit 2.
- [ ] Reserved flags for 10/11/12 exit 3 with pointer message.

## Validation

```bash
npm test -- src/cli/commands/scan src/report
node dist/cli/index.js scan fixtures/static-html --format json | node -e "JSON.parse(require('fs').readFileSync(0))"
```

## Dependencies

04, 06, 07, 08.

## Non-goals

Markdown (10), SARIF (11), baseline (12), fix logic (13+).

## Design References

DESIGN.md §8.3, §12.1 (console/json), §12.3, §17 (scan budget).
