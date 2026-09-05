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
- Registry entry + catalog message for `runtime/probe/flow-failed`
  (moderate, `bestPractice: true`, `fixability: none`, source tool
  `a11y-bot-probe`).
- Fixture pages/flows under `fixtures/flows/` and the keyboard-only example
  flow documented in the module docstring.

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
2. Flow-level semantics (DESIGN §11.5): each configured flow runs exactly once
   against the provisioned target named by `flows[].target`, in a fresh page of
   the context for `flows[].viewport` (default: the first configured
   viewport); flows run after the probes for that target, sequentially in
   config order.
3. Execution: steps sequential; each step wrapped with: start/finish
   timestamps, the acting element's descriptor
   `{ selector (as configured), tag, id?, testId?, bbox }` for
   click/fill/expect* steps, console errors emitted during the step, and an
   automatic full-page failure screenshot named `step-NN-failure.png`.
4. Failure semantics (normative): first failing step → flow status `failed`,
   finding `runtime/probe/flow-failed` with
   `{ flow, step index, step type, reason }` in messageParams + evidenceRefs;
   remaining steps skipped; other flows still run. Step timeout uses the
   payload/table defaults — no global override in v1.
5. Evidence via the EvidenceSink interface (issue 22 — scrubbing and path
   validation are sink-enforced, T7/T11), under the bundle layout of DESIGN
   §11.6: `targets/<target>/<viewport>/flows/<flow>/step-NN.json`,
   `…/step-NN.png` (explicit `screenshot` steps), `…/step-NN-failure.png`,
   and `…/summary.json` (status, durationMs, stepsExecuted, stepsTotal).
   `writeScreenshot` returning null (global `audit.maxScreenshots` cap, sink-
   enforced) is recorded in the step JSON as `screenshot: "skipped-cap"`.
6. `fill` values come from config (trusted); still masked in evidence when the
   selector or a nearby label matches `/(password|token|secret|otp)/i` → value
   recorded as `***` in step JSON (defense in depth for evidence sharing,
   §14.2 T7). Steps that filled masked fields call the sink's
   `markFlowSensitive(flow)` (issue-22 interface) so the manifest (written by
   issue 26) can flag `containsSensitiveInput: true` and users/CI can exclude
   those screenshots from artifact upload.
7. Keyboard-only journey pattern documented in the module docstring + example
   config (goto → repeated `press: Tab` → `expectFocus` → `press: Enter` →
   `waitFor`), used by a fixture test.

## Acceptance Criteria

- [ ] Happy-path flow on fixture site: all steps pass, evidence files at the
      exact §11.6 paths incl. summary.json.
- [ ] Each failure mode tested: selector timeout, expectFocus mismatch,
      expectVisible timeout, navigation error → correct finding + skip + other
      flows continue; `flow-failed` registry entry + message present.
- [ ] Origin escape attempt (`goto: https://example.com`) rejected at runtime
      (belt) and by schema (suspenders) — both tested.
- [ ] Sensitive-fill masking + `markFlowSensitive` call verified (stub sink);
      screenshot-cap `skipped-cap` recording verified.
- [ ] Security (T7/T11): step JSON URLs scrubbed via sink; hostile flow/step
      derived names impossible by schema (negative schema test).
- [ ] Viewport/target resolution semantics tested (named viewport, default
      viewport, flow order).
- [ ] Keyboard-only fixture flow passes and produces the documented evidence
      sequence.

## Validation

```bash
npm test -- src/audit/flows
```

## Dependencies

02, 21, 22 (browser session + EvidenceSink interface).

## Non-goals

Conditionals/loops/variables in the DSL (v2); multi-target flows; auth helpers.

## Design References

DESIGN.md §11.5, §14.2 T7/T9; ADR-005.
