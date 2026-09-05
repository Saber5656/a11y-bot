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
  `src/report/json.ts`, `schemas/scan-report.schema.json`. Consumes
  `applyRuleControls()` from issue 05.

## Detailed Requirements

1. CLI: `a11y-bot scan [paths...] [--format console|json|markdown|sarif]…
   [--output <dir>] [--fail-on critical|serious|moderate|minor] [--strict]
   [--update-baseline]` — markdown/sarif/baseline flags parse now but error
   "not supported yet (lands in issue 10/11/12)" as a **usage error (exit 2)**
   until those land, so the flag surface is stable from day one.
2. Format resolution: `--format` is repeatable; any `--format` REPLACES config
   `report.formats` entirely. Output routing (deterministic): `console` →
   stdout; file formats (json/markdown/sarif) → `<outputDir>/scan.<ext>`
   (`--output` > config `report.outputDir` > default `.a11ybot/reports`) —
   EXCEPT when json is the only requested format and no `--output` was given,
   json goes to stdout (piping convenience).
3. Orchestration: load config → discover (05) → run profiles (06–08 registered)
   → collect findings → `applyRuleControls` (issue 05; `off` drops, `warn` →
   severity minor + never gates, `error` keeps registry severity and gates) →
   sort findings (file, line, ruleId) → report → gate.
4. Console reporter: per-file sections `<path>:<line>:<col>  <severity>
   <ruleId>  <message>`, severity-colored, summary block (counts by severity,
   files scanned, duration). `--quiet` prints summary only.
5. JSON reporter: `{ schemaVersion: 1, run: { runId, tool: {name, version},
   startedAt, durationMs, configHash, llm?: { calls, inputTokens,
   outputTokens, rejectedOutputs } }, findings: Finding[], summary: { total,
   bySeverity, byRuleId, gatingCount } }` (`run.llm` populated by LLM-enabled
   commands from issue 18's budget; absent otherwise); to stdout by default, to
   `<outputDir>/scan.json` when `--output`/config set. This issue OWNS the
   envelope schema `schemas/scan-report.schema.json` (referencing issue 03's
   `finding.schema.json` via `$ref`); generation + CI staleness check as in
   issue 02; reporter output validated against it in tests.
6. Gate: exit 1 iff any finding has `severity ≥ failOn` AND not `advisory` AND
   not downgraded to `warn` (baseline filter slot returns all findings for now).
   `failOn` default from config (`serious`).
7. Performance: parallelize profiles with `Promise.all`; budget test fixture
   (200 files) must complete < 10 s in CI (soft assertion logged, not failing).

## Acceptance Criteria

- [ ] On fixture repo with known violations: correct findings count, ordering,
      and exit 1; `--fail-on critical` with no critical findings → exit 0.
- [ ] `scan.rules` `off|warn|error` behaviors verified end-to-end.
- [ ] JSON output validates against schema; stdout stays machine-clean
      (diagnostics on stderr) — piped test.
- [ ] `[paths...]` narrowing works and non-existent path → ConfigError exit 2.
- [ ] Reserved flags for 10/11/12 exit 2 (usage error) with pointer message.
- [ ] Format routing tested: multi-format run writes files to outputDir with
      console on stdout; json-only-no-output goes to stdout.
- [ ] T12 guards hold end-to-end through the command: `scan.maxFiles` overflow
      → exit 2; >1 MiB fixture skipped with finding; worker-timeout
      InternalError → exit 3 (integration tests reusing issue-05 hooks).

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

DESIGN.md §8.3, §12.1 (console/json), §12.3, §15 (error taxonomy), §14.2 T12,
§17 (scan budget).
