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

1. Types exactly as DESIGN.md §7.1 (field names, optionality). Provide
   `createFinding(input): Finding` that: validates rule id exists in registry,
   fills severity/wcag/fixability from registry unless explicitly overridden,
   resolves `message` from catalog (fallback to `input.rawMessage`), computes
   fingerprint, truncates/sanitizes `snippet` (≤ 200 chars, control chars
   stripped, newlines collapsed).
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
     source: { tool: string; ruleId: string };
   }
   ```
   - `registerRules(RuleMeta[])` (called by adapters/probes at module init),
     duplicate unified id → InternalError at startup.
   - `getRule(ruleId)`, `allRules()`; unknown id in `createFinding` →
     InternalError (bugs surface loudly).
   - Registry data for `runtime/axe/*` is generated lazily: axe rule ids map 1:1
     with severity from axe impact at finding time; registry holds a single
     template entry (`runtime/axe/*`) — document this exception in code.
3. Fingerprint: implement §7.3 exactly (sha256, `\0` separators, 32-hex-char
   truncation, occurrenceIndex semantics, snippet normalization = collapse
   `\s+`→single space, trim, ≤ 200 chars). Property: editing unrelated lines
   above a finding must not change its fingerprint (test with fixture).
4. Message catalog: `src/core/messages/en.ts` — `Record<messageId, string>` with
   `{placeholder}` interpolation; missing id → fallback + debug log. Catalog keys
   are added by later issues; this issue ships the mechanism plus entries for
   `bot.file-skipped`, `bot.parse-error`.
5. Severity normalization helpers per §7.4 (axe impact map; probe fixed map).
6. JSON Schema for the `json` reporter contract (`schemas/finding.schema.json`)
   generated from zod mirrors of the types; CI staleness check as in issue 02.

## Acceptance Criteria

- [ ] Type-level: `Finding` compiles matching §7.1; exported from package root.
- [ ] Fingerprint stability tests: line-shift invariance; duplicate-snippet
      disambiguation via occurrenceIndex; selector normalization for runtime
      (id present → nth-child stripped).
- [ ] Registry rejects duplicate ids and unknown lookups with InternalError.
- [ ] `createFinding` fills defaults from registry and validates WCAG refs
      (`sc` matches `^\d\.\d{1,2}\.\d{1,2}$`, no `4.1.1` allowed — test).
- [ ] Message interpolation + fallback behavior tested.
- [ ] `schemas/finding.schema.json` committed and fresh in CI.

## Validation

```bash
npm test -- src/core
npm run gen:schemas && git diff --exit-code schemas/
```

## Dependencies

01.

## Non-goals

Any concrete rule tables (adapters own them); baseline persistence (12); SARIF
mapping (11).

## Design References

DESIGN.md §7 (all), §3 (WCAG metadata, 4.1.1 exclusion), §12.1 (json contract).
