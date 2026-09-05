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
   `<audit.evidenceDir>/<runId>/`; all path components validated: target names
   and viewport names are schema-constrained (issue 02: `^[a-z0-9-]{1,40}$` /
   `^[a-z0-9-]{1,20}$`), and every sink-relative path must match
   `^[a-z0-9/._-]+\.(json|png)$` with no `..` segments (assert; §14.2 T11).
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
3. Scrubbing (applies at write time to EVERY JSON string field recursively and
   to file names): URL-shaped strings are URL-parsed and query-param values
   whose key case-insensitively matches any `audit.scrubParams` entry → `***`;
   this covers page URLs, console/message/request-URL records, DOM-derived
   snippets and outline text fields (any string containing a `?key=` pair is
   scrub-checked even outside a full URL).
4. Caps: total screenshots ≤ `audit.maxScreenshots` (`writeScreenshot` returns
   null beyond; manifest `truncated: true` + count of skipped); single JSON
   file ≤ 2 MiB — oversized arrays are truncated from the tail and a final
   element `{ "__truncated": true, "omitted": <n> }` is appended (applies to
   any array field; non-array overflow → InternalError); whole-bundle soft cap
   100 MiB → warning log.
5. Reader API for 28/10: `loadManifest(dir)` and `loadTargetEvidence(dir,
   target, viewport)` returning
   `{ axe?, outline?, focusOrder?, probes?, console?, flows: Record<string,
   { summary, steps }> }` — each field typed via zod schemas mirroring the
   `schemas/` files; a missing optional file → `undefined` field; malformed
   present file → InternalError; manifest schema violations → InternalError.
6. Retention: export `pruneOldRuns(evidenceDir, keepRuns)` — deletes oldest
   run directories beyond `audit.keepRuns` (0 = keep all; key exists in the
   schema from issue 02). The audit command (issue 27) CALLS this after a
   successful run — no command wiring here.
7. Nothing in the bundle is ever written outside `evidenceDir` (assert in
   writer; test the assertion). U7 closure: verify the current GitHub Actions
   artifact size limits from official docs during implementation and record
   the number + source URL in a code comment AND update the DESIGN §2.3 U7
   row (acceptance-checked).

## Acceptance Criteria

- [ ] Full audit fixture run produces the exact layout; manifest validates
      against schema; staleness check in CI.
- [ ] Scrub tests: `?token=abc&page=2` → `?token=***&page=2` in page URLs,
      console records, flow step JSONs, AND outline.json DOM-derived text.
- [ ] Cap tests: screenshot cap honored + manifest truncated flag; JSON array
      truncation marker `{ "__truncated": true, "omitted": n }` present.
- [ ] Reader round-trip typed-load test; missing probe file tolerated;
      malformed manifest → InternalError.
- [ ] Retention: `pruneOldRuns` deletes oldest beyond keepRuns; `keepRuns: 0`
      keeps all (function-level tests; command wiring tested in 27).
- [ ] Security (T7/T11): write-outside-dir assertion test; default evidenceDir
      lives under `.a11ybot/` which issue 02's `init` gitignores (test asserts
      the default path prefix); `containsSensitiveInput` flag from
      `markFlowSensitive` lands in the manifest.
- [ ] U7: DESIGN §2.3 row updated with the verified artifact limit + source.

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
