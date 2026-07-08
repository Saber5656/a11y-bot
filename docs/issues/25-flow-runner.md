# Title

Scripted flow runner (config DSL) with per-step evidence

## Summary

Implement the closed-vocabulary flow DSL from config (`audit.flows[]`): step
execution, assertion semantics, failure handling, and per-step evidence capture.

## Context

DESIGN.md §11.5 is normative (step vocabulary, failure semantics). Flows are how
users encode "can this journey be completed" checks deterministically; the LLM
analyst (28) later reads the step evidence to comment on UX friction. Config
shape is validated in issue 02; runtime behavior lives here.

## Scope

- `src/audit/flows.ts`.

## Detailed Requirements

1. Step vocabulary (closed set, anything else is already a ConfigError from 02):
   | Step | Payload | Behavior |
   |---|---|---|
   | `goto` | path string | navigate relative to target baseUrl (absolute URLs → ConfigError at load: flows never leave the target origin — enforce origin check HERE too, §14.2 T9) |
   | `click` | selector | `locator.click({ timeout: 5000 })` |
   | `fill` | `{ selector, value }` | `locator.fill(value, { timeout: 5000 })` |
   | `press` | key string (Playwright key syntax) | `page.keyboard.press` |
   | `waitFor` | `{ selector, timeoutMs? (default 5000, max 30000) }` | wait for visible |
   | `screenshot` | name `[a-z0-9-]{1,40}` | full-page screenshot to evidence |
   | `expectFocus` | selector | assert `document.activeElement` matches selector |
   | `expectVisible` | selector | assert locator visible |
2. Execution: steps sequential; each step wrapped with: start/finish timestamps,
   the acting element's descriptor (click/fill/expect*), console errors emitted
   during the step, and an automatic small screenshot on failure.
3. Failure semantics (normative): first failing step → flow status `failed`,
   finding `runtime/probe/flow-failed` (moderate) with
   `{ flow, step index, step type, reason }` in message + evidenceRefs;
   remaining steps skipped; other flows still run. Step timeout uses the
   payload/table defaults — no global override in v1.
4. Evidence: per step JSON `flows/<flow>/step-NN.json` + screenshots
   `step-NN.png` (failure or explicit `screenshot` step) via EvidenceSink;
   flow summary JSON (status, duration, steps executed).
5. `fill` values come from config (trusted); still masked in evidence when the
   selector or a nearby label matches `/(password|token|secret|otp)/i` → value
   recorded as `***` in step JSON (defense in depth for evidence sharing,
   §14.2 T7); screenshots after such steps are taken BEFORE the fill renders?
   — not reliably possible: instead, steps that filled masked fields mark the
   flow's subsequent screenshots with `containsSensitiveInput: true` in the
   manifest so users/CI can exclude them from artifact upload (document in 26
   manifest schema; implement flag here).
6. Keyboard-only journey pattern documented in the module docstring + example
   config (goto → repeated `press: Tab` → `expectFocus` → `press: Enter` →
   `waitFor`), used by a fixture test.

## Acceptance Criteria

- [ ] Happy-path flow on fixture site: all steps pass, evidence files complete.
- [ ] Each failure mode tested: selector timeout, expectFocus mismatch,
      expectVisible timeout, navigation error → correct finding + skip + other
      flows continue.
- [ ] Origin escape attempt (`goto: https://example.com`) rejected at runtime
      (belt) and by schema (suspenders) — both tested.
- [ ] Sensitive-fill masking + screenshot flag behavior tested.
- [ ] Keyboard-only fixture flow passes and produces the documented evidence
      sequence.

## Validation

```bash
npm test -- src/audit/flows
```

## Dependencies

21, 02 (+ session from 22 at integration level).

## Non-goals

Conditionals/loops/variables in the DSL (v2); multi-target flows; auth helpers.

## Design References

DESIGN.md §11.5, §14.2 T7/T9; ADR-005.
