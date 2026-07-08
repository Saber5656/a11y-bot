# Title

Unified finding model, rule registry, fingerprints, message catalog

## Summary

Implement the core data model every engine and reporter shares: `Finding`,
`WcagRef`, severity/fixability enums, the rule registry, the fingerprint
algorithm, and the English message catalog.

## Context

DESIGN.md §7 is normative. Adapters (06–08), probes (23–24), reporters (09–11),
baseline (12), and the fix engine (13) all consume these types; getting them right
first prevents churn.

## Scope

- `src/core/findings.ts` (types + constructors/validators).
- `src/core/registry.ts` (rule registry structure + lookup API).
- `src/core/fingerprint.ts`.
- `src/core/messages/` (catalog + resolver).
- `schemas/finding.schema.json` generation (same mechanism as issue 02).

## Detailed Requirements

1. Types exactly as DESIGN.md §7.1 (field names, optionality). Constructors:
   ```ts
   interface CreateFindingInput {
     ruleId: string;
     // static engine:
     file?: string; range?: Range; sourceText?: string;  // snippet derived from
                                                         // sourceText at range
     // runtime/ux engines:
     target?: Finding["target"]; evidenceRefs?: string[];
     // dynamic-namespace rules only (see req. 2): required there, else forbidden:
     severity?: Severity; wcag?: WcagRef[];
     // message resolution:
     messageParams?: Record<string, string>; rawMessage?: string;
     occurrenceIndex?: number;            // default 0 (see req. 3)
     advisory?: true;
   }
   createFinding(input: CreateFindingInput): Finding
   createFindings(inputs: CreateFindingInput[]): Finding[]   // batch, see req. 3
   ```
   `createFinding` validates rule id against the registry, fills
   severity/wcag/fixability/`source` from `RuleMeta`, resolves `message` from
   the catalog (fallback `rawMessage`), computes the fingerprint, and derives
   `snippet` from `sourceText` (≤ 200 chars, control chars stripped, newlines
   collapsed).
2. Registry:
   ```ts
   interface RuleMeta {
     ruleId: string;                     // unified id (§7.2 grammar)
     engine: Engine;
     wcag: WcagRef[];                    // empty only if bestPractice
     bestPractice?: true;
     defaultSeverity: Severity;
     fixability: Fixability;
     messageId: string;
     docsUrl?: string;
     source: { tool: string; version: string; ruleId: string };
     // version = the upstream package version, supplied by the registering
     // adapter (adapters read their plugin's package.json version once).
     configGated?: string;               // e.g. "fix.defaults.lang" (issue 06/08)
   }
   ```
   - `registerRules(RuleMeta[])` (called by adapters/probes at module init),
     duplicate unified id → InternalError (class from issue 01) at startup.
   - `getRule(ruleId)`, `allRules()`; unknown id in `createFinding` →
     InternalError (bugs surface loudly).
   - **Dynamic namespaces** (DESIGN §7.2 exception):
     `registerDynamicNamespace(prefix, template)` for `runtime/axe/` and
     `ux/llm/` where `template` fixes `engine`, `fixability`, `source.tool`;
     for findings under a dynamic namespace, `severity` and `wcag` are REQUIRED
     on `CreateFindingInput` (validated; `wcag: []` allowed with
     `advisory: true` or template `bestPractice`). Non-dynamic rules reject
     input-level severity/wcag overrides.
3. Fingerprint: implement §7.3 exactly (sha256, `\0` separators, 32-hex-char
   truncation, snippet normalization = collapse `\s+`→single space, trim,
   ≤ 200 chars). `occurrenceIndex` semantics: `createFindings(inputs)` computes
   it per batch by grouping on (file ?? target.name, ruleId, normalizedSnippet)
   in input order (0-based); callers that construct findings one-by-one (probes)
   pass an explicit index or accept the default 0. Property: editing unrelated
   lines above a finding must not change its fingerprint (test with fixture).
4. Message catalog: `src/core/messages/en.ts` — `Record<messageId, string>` with
   `{placeholder}` interpolation; missing id → fallback + debug log. Catalog keys
   are added by later issues; this issue ships the mechanism plus entries for
   `bot.file-skipped`, `bot.parse-error`.
5. Severity normalization helpers per §7.4 (axe impact map; probe fixed map).
   Plus the vendored WCAG SC list: `src/core/wcag-sc-list.ts` exporting the
   set of all WCAG 2.0/2.1/2.2 Level A and AA SC numbers with their
   level/version metadata (source: W3C quickref; source URL in comment) —
   used by WcagRef validation here and by the analyst validator (issue 28).
6. JSON Schema for the **Finding object only** (`schemas/finding.schema.json`)
   generated from zod mirrors of the types; CI staleness check as in issue 02.
   The full report envelope schema (`schemas/scan-report.schema.json`,
   `{ schemaVersion, run, findings, summary }` per DESIGN §12.1) is owned by
   issue 09 and references this Finding schema.

## Acceptance Criteria

- [ ] Type-level: `Finding` compiles matching §7.1; exported from package root.
- [ ] Fingerprint stability tests: line-shift invariance; duplicate-snippet
      disambiguation via occurrenceIndex; selector normalization for runtime
      (id present → nth-child stripped).
- [ ] Registry rejects duplicate ids and unknown lookups with InternalError;
      dynamic-namespace behavior tested (severity/wcag required there,
      rejected elsewhere).
- [ ] `createFinding` fills defaults from registry and validates WCAG refs:
      `sc` matches `^\d\.\d{1,2}\.\d{1,2}$`, `level` ∈ {A, AA}, `version` ∈
      {2.0, 2.1, 2.2}, no `4.1.1` allowed, and `wcag: []` only with
      bestPractice/advisory (all tested).
- [ ] Message interpolation + fallback behavior tested; interpolated params are
      sanitized (newlines/control chars stripped) so hostile file content
      cannot forge extra log/report lines (DESIGN §14.2 T1 surface).
- [ ] Snippet sanitization tested: ANSI/control sequences in `sourceText` never
      reach `snippet` or `message`.
- [ ] `schemas/finding.schema.json` committed and fresh in CI.

## Validation

```bash
npm test -- src/core
npm run gen:schemas && git diff --exit-code schemas/
```

## Dependencies

01 (error classes from `src/core/errors.ts`).

## Non-goals

Any concrete rule tables (adapters own them); baseline persistence (12); SARIF
mapping (11); the report envelope schema (09).

## Design References

DESIGN.md §7 (all, incl. §7.2 dynamic namespaces), §3 (WCAG metadata, 4.1.1
exclusion), §12.1 (json contract split), §14.2 T1/T3.
