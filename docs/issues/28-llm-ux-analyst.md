# Title

LLM UX analyst over evidence bundle (advisory findings)

## Summary

Implement the optional LLM pass that reads the evidence bundle and produces
schema-constrained, advisory UX findings rendered in audit reports.

## Context

DESIGN.md §10.3 (analyst) and ADR-005: the browser is closed before the model
runs; input is only the deterministic evidence. Output is advisory — never gates
CI, never enters the baseline (12.6).

## Scope

- `src/llm/analyst.ts` + prompt assembly + report integration hook (27's stub).

## Detailed Requirements

1. Input assembly per target×viewport (budget-aware, at most one call per
   target×viewport; targets beyond budget skipped with notice):
   - manifest excerpt (target, viewport, page URL scrubbed);
   - axe summary (rule id, count, top selectors — max 30 lines);
   - outline.json (structure probe);
   - focus-order walk summary (sequence of role/name, visibility flags);
   - probe findings list (compact);
   - flow summaries incl. failure step data;
   - up to 5 screenshots (full-page first, then failure/step shots; flows with
     `containsSensitiveInput: true` excluded from screenshot selection);
   - ALL text framed via `frameUntrusted` (19) with source labels.
2. Output JSON schema (constrained via client `jsonSchema`):
   ```json
   { "findings": [ { "category": "navigation|forms|structure|media|feedback|motion|other",
       "title": "≤ 80 chars", "description": "≤ 500 chars",
       "severityHint": "advisory|low|medium|high",
       "wcagRefs": ["2.4.7"], "evidenceRefs": ["targets/…/screenshots/full.png"],
       "confidence": 0.0 } ] }
   ```
   Max 20 findings per call (schema `maxItems`).
3. Validation pipeline per finding: schema (client) → `validateLlmText` on
   title/description (prose kind) → `wcagRefs` must match SC regex and exist in
   a vendored SC list (else dropped) → `evidenceRefs` must exist in the bundle
   (reader from 26; else ref dropped, finding kept if ≥ 0 refs remain? NO —
   finding kept only if at least one valid ref remains OR category `structure`
   with outline-based rationale; otherwise dropped) → confidence < 0.3 dropped.
   Dropped counts surface as `llm.rejectedOutputs` (19.6).
4. Accepted findings → unified findings: ruleId `ux/llm/<category>`,
   `advisory: true`, severity mapped `advisory|low → minor`, `medium →
   moderate`, `high → serious` (display only — advisory flag excludes gating),
   engine `ux`, target set, evidenceRefs preserved.
5. Rendering: markdown report's "Advisory UX findings (AI-assisted)" section
   (10) with confidence column; JSON report includes them with `advisory: true`.
6. Failure containment: LlmUnavailable / all-rejected → audit completes
   normally with a one-line notice (never exit-code impact). Wall-clock cap:
   analyst phase ≤ 3 min (constant) then abandoned with notice.

## Acceptance Criteria

- [ ] Stub-provider integration test: canned evidence bundle → expected
      advisory findings in JSON + markdown outputs.
- [ ] Validation drops: bad wcagRef, nonexistent evidenceRef, low confidence,
      hostile description (adversarial corpus reuse) — each tested.
- [ ] Sensitive-flow screenshot exclusion tested.
- [ ] Budget: 3 targets, maxCalls=2 → third skipped with notice.
- [ ] No-key mode: audit output identical to analyst-disabled run (snapshot).

## Validation

```bash
npm test -- src/llm/analyst
```

## Dependencies

26, 19.

## Non-goals

Gating on analyst output (never); cross-run trend analysis; auto-fix from UX
findings (v2); analyst-driven browser actions (ADR-005).

## Design References

DESIGN.md §10.3, §10.2, §11.6; ADR-004, ADR-005.
