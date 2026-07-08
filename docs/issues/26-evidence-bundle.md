# Title

Evidence bundle: layout, manifest, scrubbing, caps

## Summary

Implement the on-disk evidence bundle (the `EvidenceSink` interface defined in
issue 22): directory layout, versioned manifest, URL/param scrubbing, size caps,
and the read API used by the LLM analyst and reporters.

## Context

DESIGN.md §11.6 is normative. The bundle is the contract between the
deterministic audit and everything downstream (LLM analyst 28, CI artifact
upload, human debugging) — ADR-005's abstraction boundary.

## Scope

- `src/audit/evidence.ts` (writer implementing EvidenceSink + reader),
  `schemas/evidence-manifest.schema.json`.

## Detailed Requirements

1. Layout exactly per DESIGN §11.6 under
   `<audit.evidenceDir>/<runId>/`; all path components validated
   (`targets/<name>/<viewport>/…`, names already constrained by config schema;
   screenshot names generated internally — assert `^[a-z0-9-]+(\.png|\.json)$`,
   §14.2 T11).
2. `manifest.json`:
   ```json
   { "schemaVersion": 1, "runId": "…", "tool": { "name": "a11y-bot",
     "version": "…" }, "startedAt": "…", "durationMs": 0,
     "configHash": "…", "targets": [ { "name": "…", "baseUrl": "…",
     "viewports": ["desktop","mobile"], "pages": [...], "flows": [
       { "name": "…", "status": "passed|failed",
         "containsSensitiveInput": false } ] } ],
     "probes": ["axe","focus-order", "..."],
     "counts": { "screenshots": 0, "findings": 0 },
     "truncated": false }
   ```
   JSON Schema committed + staleness-checked like other schemas.
3. Scrubbing (applies at write time to every JSON string field and file name):
   query-param values whose key case-insensitively matches any
   `audit.scrubParams` entry → `***` (URL-parse based, not regex on whole
   string); the same scrub applies to console/message/request-URL records from
   the session taps.
4. Caps: total screenshots ≤ `audit.maxScreenshots` (writer returns
   `skipped: true` beyond; manifest `truncated: true` + count of skipped);
   single JSON file ≤ 2 MiB (truncate arrays with marker); whole-bundle soft
   cap 100 MiB → warning log (known unknown U7: record the practical Action
   artifact limit in code comment + DESIGN §2.3 update in this PR).
5. Reader API for 28/10: `loadManifest(dir)`, `loadTargetEvidence(dir, target,
   viewport)` returning typed structures; tolerant of missing optional files
   (probe skipped) but strict on manifest schema.
6. Retention/cleanup: `a11y-bot audit` keeps the last 3 runs under evidenceDir
   (delete older, oldest-first; disabled via `audit.keepRuns: 0` meaning
   keep-all — ADD `keepRuns` (int ≥ 0, default 3) to schema + DESIGN §6.2 in
   this PR).
7. Nothing in the bundle is ever written outside `evidenceDir` (assert in
   writer; test with hostile-ish names is moot given generation-side
   constraints — still test the assertion).

## Acceptance Criteria

- [ ] Full audit fixture run produces the exact layout; manifest validates
      against schema; staleness check in CI.
- [ ] Scrub tests: `?token=abc&page=2` → `?token=***&page=2` in page URLs,
      console records, and flow step JSONs.
- [ ] Cap tests: screenshot cap honored + manifest truncated flag; JSON
      truncation marker present.
- [ ] Reader round-trip typed-load test; missing probe file tolerated.
- [ ] Retention: 4th run deletes the oldest; `keepRuns: 0` keeps all.

## Validation

```bash
npm test -- src/audit/evidence
```

## Dependencies

22, 23, 24, 25 (interface consumers exist to integration-test against).

## Non-goals

Uploading artifacts (user workflows / templates, 32); evidence diffing (v2).

## Design References

DESIGN.md §11.6, §2.3 U7, §14.2 T7/T11; ADR-005.
