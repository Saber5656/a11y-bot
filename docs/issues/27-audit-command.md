# Title

`audit` command: orchestration, reports, gating

## Summary

Wire provisioning, session, axe, probes, flows, and evidence into
`a11y-bot audit` with reports and baseline-aware gating.

## Context

The runtime counterpart of issue 09. DESIGN §11 pipeline + §12 gating.
Deterministic findings (axe + probes) may gate; advisory (ux/llm, axe
incomplete) never gate.

## Scope

- `src/cli/commands/audit.ts` (replace stub).

## Detailed Requirements

1. CLI: `a11y-bot audit [--target <name>...] [--format …] [--output <dir>]
   [--fail-on <sev>] [--strict] [--update-baseline] [--llm] [--no-evidence]`.
   - `--target` filters configured targets (unknown name → exit 2);
   - zero configured targets → ConfigError with hint to add `audit.targets`;
   - `--no-evidence` runs probes but writes only manifest+JSON (no screenshots)
     — for constrained CI.
2. Orchestration order per target: provision → per viewport: [load page(s) →
   axe → probes 23/24] → flows (25) → teardown; pages = target base path only
   in v1 (multi-path via flows' `goto`) — document explicitly.
   Concurrency: targets sequential (deterministic logs); viewports sequential.
   Whole-run wall-clock budget: warn at 15 min (constant).
3. Findings pipeline: collect → dedupe (24's axe-suppression map) → baseline
   subtraction (12, same file — fingerprints are engine-agnostic) → reports
   (console/json/markdown; SARIF excluded for runtime — issue 11 rule) →
   exit gate identical semantics to 09/12.
4. LLM analyst (28) invoked after evidence completion when enabled
   (`--llm` or config + key); its advisory findings merge into reports but
   never the gate; when 28 not yet implemented, the hook logs "analyst not
   available" (interface stub) — this issue lands independently.
5. `--update-baseline` covers runtime findings too (shared mechanism; narrowed
   `--target` runs skip stale pruning, mirroring 12's path rule).
6. Summary console block: targets audited, pages, flows passed/failed,
   findings by severity (new/known), evidence dir path, analyst status.
7. Exit codes: 1 gate, 2 config, 3 internal, 4 all-targets-failed or browser
   missing (from 21/22 semantics).

## Acceptance Criteria

- [ ] Fixture static site end-to-end: findings from axe+probes+failed flow,
      evidence bundle complete, exit 1; after `--update-baseline` → exit 0.
- [ ] `--target` filtering, unknown target error, zero-target hint tested.
- [ ] Advisory findings present in reports but never gate (fixture with only
      advisory → exit 0).
- [ ] `--no-evidence` writes no PNGs but manifest notes the mode.
- [ ] Partial target failure (1 of 2 unreachable) → exit reflects surviving
      target's gate; failure recorded as finding + summary line.

## Validation

```bash
npx playwright install chromium
npm test -- src/cli/commands/audit
node dist/cli/index.js audit --target built --format markdown --output .a11ybot/reports   # in fixtures/static-html
```

## Dependencies

26, 10, 12.

## Non-goals

Analyst implementation (28); PR creation from audit (v1 audits never write
code); crawling.

## Design References

DESIGN.md §11 (all), §12.1–12.3; ADR-001 (runtime = report/gate only), ADR-005.
