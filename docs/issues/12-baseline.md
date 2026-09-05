# Title

Baseline file and fail-on-new gating

## Summary

Implement the baseline mechanism that freezes existing findings so CI fails only
on new ones, plus `--update-baseline` and staleness reporting.

## Context

DESIGN.md §12.2. The gate seam was left in issue 09; this issue fills it. The
fingerprint algorithm (03) was designed for line-drift stability specifically for
this feature.

## Scope

- `src/report/baseline.ts` (load/save/diff), `scan` integration including the
  `--update-baseline` flag behavior, `--strict`.

## Detailed Requirements

1. File format (`baseline.file`, default `.a11ybot/baseline.json`) — exactly
   the DESIGN §12.2 shape:
   ```json
   { "schemaVersion": 1, "createdAt": "<ISO8601>",
     "fingerprints": ["<32hex>", "..."] }
   ```
   Sorted, deduplicated fingerprints; file ends with newline (clean diffs).
   By construction the file contains ONLY hex fingerprints + metadata — a
   schema (`schemas/baseline.schema.json`, staleness-checked) enforces this,
   which is also the security property (no snippets/paths/secrets can enter).
2. Load: missing file → empty baseline (all findings new); malformed →
   ConfigError with hint to re-run `--update-baseline`; unknown schemaVersion →
   ConfigError.
3. Gate integration (replaces issue 09 pass-through): finding gates iff
   `severity ≥ failOn` AND `!advisory` AND `fingerprint ∉ baseline` AND rule
   control ≠ warn/off. `--strict` skips baseline subtraction entirely.
4. Classification exposed to reporters: each finding annotated
   `baselineStatus: "new" | "known"`; summary gains
   `{ newCount, knownCount, staleCount }` (stale = baseline fingerprints not
   observed this run — only counted when the run's scope covers the full include
   set, i.e., no `[paths...]` narrowing; narrowed runs print "stale check
   skipped (narrowed run)").
5. `--update-baseline`: writes fingerprints of all current **non-advisory**
   findings (severity-independent; advisory/ux excluded per requirement 6),
   prunes stale entries, prints delta summary (+N added, −M pruned). Exits 0
   even when findings exist (its purpose is adoption). Combination with
   `--strict` → ConfigError (contradictory).
6. Advisory/ux findings never enter the baseline (they are non-gating by
   definition; keeping them out avoids fingerprint churn from model variance).
7. Reporter integration: findings are annotated `baselineStatus` (§7.1); the
   SARIF exporter emits it as `properties.baselineStatus` (issue 11 hook).

## Acceptance Criteria

- [ ] Adoption flow test: scan (exit 1) → `--update-baseline` (exit 0) → scan
      (exit 0) → introduce new violation → scan (exit 1, `newCount: 1`).
- [ ] Line-drift test: insert 5 lines above a known violation → still `known`.
- [ ] Stale pruning + narrowed-run skip behaviors tested.
- [ ] `--strict` ignores baseline; `--strict --update-baseline` → exit 2.
- [ ] Markdown/JSON/console reporters show new/known counts and SARIF results
      carry `properties.baselineStatus` (extend issue 09/10/11 outputs;
      snapshots updated).
- [ ] Security/schema: baseline file validates against
      `schemas/baseline.schema.json`; a file with a non-hex entry is rejected
      with ConfigError; env-canary test shows no secret values can round-trip
      through the baseline.

## Validation

```bash
npm test -- src/report/baseline
npm run gen:schemas && git diff --exit-code schemas/
bash -eux <<'E2E'
cd "$(mktemp -d)" && cp -r "$OLDPWD/fixtures/static-html" app && cd app
A11Y=$OLDPWD/dist/cli/index.js
node "$A11Y" scan . && exit 1 || test $? -eq 1        # violations gate
node "$A11Y" scan . --update-baseline                 # adopt (exit 0)
node "$A11Y" scan .                                   # clean now (exit 0)
printf '<img src="x.png">' >> index.html
node "$A11Y" scan . && exit 1 || test $? -eq 1        # new violation gates
E2E
```

## Dependencies

09.

## Non-goals

Audit-command gating wiring (27 reuses this module); trend history (v2).

## Design References

DESIGN.md §12.2, §7.3 (fingerprint), §12.3.
