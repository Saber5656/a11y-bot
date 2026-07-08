# Title

LLM UX analyst over evidence bundle (advisory findings)

## Summary

Implement the optional LLM pass that reads the evidence bundle and produces
schema-constrained, advisory UX findings rendered in audit reports.

## Context

DESIGN.md §10.3 (analyst) and ADR-005: the browser is closed before the model
runs; input is only the deterministic evidence. Output is advisory — never
gates CI, never enters the baseline (DESIGN §12.2 / issue 12 requirement 6).

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
   title/description (prose kind) → `wcagRefs` must match the SC regex and
   exist in the vendored SC list `src/core/wcag-sc-list.ts` (issue 03; invalid
   refs dropped from the array) → every `evidenceRefs` entry must exist in the
   bundle (reader from 26; nonexistent refs dropped) → the finding is KEPT only
   if at least one valid evidenceRef remains (no exceptions) AND
   confidence ≥ 0.3. Dropped findings increment the issue-19 requirement-6
   counter (`rejectedOutputs`), surfaced via `run.llm` in the report envelope
   (issues 18/09).
4. Accepted findings → unified findings via `createFinding` (issue 03 dynamic
   namespace `ux/llm/`): ruleId `ux/llm/<category>`, `engine: "ux"`,
   `advisory: true`, `confidence` set, severity input mapped
   `advisory|low → minor`, `medium → moderate`, `high → serious` (display
   only), `wcag` = validated refs (empty allowed — advisory), `fixability:
   "none"`, `source` = the dynamic-namespace template
   `{ tool: "a11y-bot-analyst", version: <package version>, ruleId:
   <category> }`, `target` = the audited target/viewport, `rawMessage` =
   validated description (message resolution), `evidenceRefs` preserved;
   fingerprint inputs per §7.3 runtime form use target name + normalized
   title as the snippet.
5. Rendering: markdown report's "Advisory UX findings (AI-assisted)" section
   (10) with confidence column; JSON report includes them with `advisory: true`.
6. Failure containment: LlmUnavailable / all-rejected → audit completes
   normally with a one-line notice (never exit-code impact). Wall-clock cap:
   analyst phase ≤ 3 min (constant) then abandoned with notice.

## Acceptance Criteria

- [ ] Stub-provider integration test: canned evidence bundle → expected
      advisory findings in JSON + markdown outputs (through the issue-27 hook
      and issue-10 renderer).
- [ ] Validation drops: bad wcagRef, nonexistent evidenceRef, zero-valid-refs,
      low confidence, hostile description (adversarial corpus reuse) — each
      tested and counted in `run.llm.rejectedOutputs`.
- [ ] Sensitive-flow screenshot exclusion tested.
- [ ] Security (DESIGN §14.2 T1/T7/T8, §10.2): assembled prompts pass
      `assertPromptClean` (env canaries, absolute paths); stub payloads pass
      `assertNoToolFields`; only bundle-local images are attached (no remote
      fetch — request log assertion); maxCalls/timeout caps enforced
      (budget test); analyst wall-clock cap abandons cleanly.
- [ ] Budget: 3 targets, maxCalls=2 → third skipped with notice.
- [ ] No-key or disabled mode: audit output identical to analyst-disabled run
      (snapshot).

## Validation

```bash
npm test -- src/llm/analyst
```

## Dependencies

10 (rendering), 19, 26, 27 (analyst hook + report integration).

## Non-goals

Gating on analyst output (never); cross-run trend analysis; auto-fix from UX
findings (v2); analyst-driven browser actions (ADR-005).

## Design References

DESIGN.md §10.3, §10.2, §11.6; ADR-004, ADR-005.
