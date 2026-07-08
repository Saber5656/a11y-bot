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

- `src/report/baseline.ts` (load/save/diff), `scan` integration, `--strict`.

## Detailed Requirements

1. File format (`baseline.file`, default `.a11ybot/baseline.json`):
   ```json
   { "schemaVersion": 1, "tool": "a11y-bot", "createdAt": "<ISO8601>",
     "fingerprints": ["<32hex>", "..."] }
   ```
   Sorted, deduplicated fingerprints; file ends with newline (clean diffs).
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
5. `--update-baseline`: writes fingerprints of **all current findings**
   (severity-independent), prunes stale entries, prints delta summary
   (+N added, −M pruned). Exits 0 even when findings exist (its purpose is
   adoption). Combination with `--strict` → ConfigError (contradictory).
6. Advisory/ux findings never enter the baseline (they are non-gating by
   definition; keeping them out avoids fingerprint churn from model variance).

## Acceptance Criteria

- [ ] Adoption flow test: scan (exit 1) → `--update-baseline` (exit 0) → scan
      (exit 0) → introduce new violation → scan (exit 1, `newCount: 1`).
- [ ] Line-drift test: insert 5 lines above a known violation → still `known`.
- [ ] Stale pruning + narrowed-run skip behaviors tested.
- [ ] `--strict` ignores baseline; `--strict --update-baseline` → exit 2.
- [ ] Markdown/JSON/console reporters show new/known counts (extend issue 09/10
      outputs; snapshots updated).

## Validation

```bash
npm test -- src/report/baseline
# scripted end-to-end in fixtures/static-html per acceptance flow above
```

## Dependencies

09.

## Non-goals

Audit-command gating wiring (27 reuses this module); trend history (v2).

## Design References

DESIGN.md §12.2, §7.3 (fingerprint), §12.3.
