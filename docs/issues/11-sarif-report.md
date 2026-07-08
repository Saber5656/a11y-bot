# Title

SARIF 2.1.0 exporter for GitHub code scanning

## Summary

Implement `--format sarif` producing SARIF 2.1.0 that GitHub code scanning
ingests, with stable fingerprints and correct severity levels.

## Context

SARIF lets findings appear as code-scanning alerts. Research doc
(runtime-audit-tooling §SARIF) fixes the mappings; DESIGN.md §12.1. Runtime/ux
findings have no file location — they are excluded from SARIF (documented).

## Scope

- `src/report/sarif.ts` (`renderSarif(findings, run): SarifLog`), wiring into
  `scan --format sarif` → `<outputDir>/scan.sarif`.

## Detailed Requirements

1. Structure: single `run`; `tool.driver = { name: "a11y-bot", version,
   informationUri, rules: ReportingDescriptor[] }` where rules are the distinct
   ruleIds present (id = unified ruleId; `helpUri` = registry docsUrl;
   `shortDescription` from catalog; `properties.tags` include `accessibility`
   and `external/wcag/<sc>` per WCAG ref).
2. Results: only findings with `file` (static engine). `level` mapping:
   critical|serious → `error`, moderate → `warning`, minor → `note`.
   `partialFingerprints: { "a11ybotFingerprint/v1": finding.fingerprint }`.
   Region uses 1-based start/end line/column from `range`.
3. `originalUriBaseIds`: emit `SRCROOT` and relative `artifactLocation.uri`
   (POSIX) + `uriBaseId: "SRCROOT"` so alerts anchor correctly regardless of
   checkout path.
4. Advisory/ux findings and runtime findings: excluded; exporter logs an info
   line with the excluded count (visible in CI logs).
5. Output validated in tests against the official SARIF 2.1.0 JSON Schema
   (vendored into `fixtures/schemas/sarif-2.1.0.json`).
6. Size guard: > 5,000 results → truncate with `runs[0].properties.truncated:
   true` and warning log (GitHub ingestion caps; exact cap re-verified during
   implementation per known unknown U3 — record the verified number in code
   comment + DESIGN.md update).

## Acceptance Criteria

- [ ] Generated SARIF validates against the vendored schema in CI.
- [ ] Severity→level and fingerprint mapping unit-tested.
- [ ] Fixture scan produces SARIF with correct relative URIs and rule metadata
      (snapshot).
- [ ] In this repo's CI, a one-off job uploads fixture SARIF via
      `github/codeql-action/upload-sarif` and succeeds (smoke; may be marked
      `continue-on-error: false` and run only on main pushes).
- [ ] U3 verification note recorded (code comment + DESIGN §2.3 row updated).

## Validation

```bash
npm test -- src/report/sarif
node dist/cli/index.js scan fixtures/static-html --format sarif --output .a11ybot/reports
npx ajv validate -s fixtures/schemas/sarif-2.1.0.json -d .a11ybot/reports/scan.sarif
```

## Dependencies

03, 09.

## Non-goals

Runtime-finding SARIF (needs URI scheme design — v2); uploading from the Action
(user workflows do it; templates in 32).

## Design References

DESIGN.md §12.1, §2.3 U3; research/2026-07-runtime-audit-tooling.md (SARIF).
