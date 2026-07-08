# Title

Probes: focus-order walk, focus visibility, keyboard reachability

## Summary

Implement the deterministic keyboard-interaction probes that record the Tab
traversal, detect invisible focus indicators, and find keyboard-unreachable
interactive elements.

## Context

DESIGN.md §11.4 rows 1–2. These probes are the heart of the "UX included"
differentiator and feed both findings and the evidence bundle consumed by the
LLM analyst. Thresholds are code constants (determinism over configurability).
Known unknown U4 (false-positive rate of the visibility heuristic) is tuned here
on fixtures.

## Scope

- `src/audit/probes/focus-order.ts`, `src/audit/probes/keyboard-reach.ts`,
  shared `src/audit/probes/interactive-census.ts`.

## Detailed Requirements

1. Interactive census (shared): collect elements matching
   `a[href], button, input:not([type=hidden]), select, textarea, summary,
   [tabindex]:not([tabindex="-1"]), [role=button], [role=link], [role=checkbox],
   [role=radio], [role=tab], [role=menuitem], [contenteditable=true]`,
   excluding `disabled`, `aria-hidden="true"` ancestors, `visibility:hidden`/
   `display:none`, and zero-area boxes. Output: descriptor list
   `{ selector (stable: prefer id → data-testid → nth-path), role, name
   (accessible name via aria-label/labelledby/text, best-effort), bbox }`.
2. focus-order probe:
   - start at `<body>`; press Tab up to `min(200, census*2)` times; after each
     stop record: active element descriptor, bbox, and **focus-visibility
     evidence**: computed styles (outline-style/width/color, box-shadow,
     border) in focused vs blurred state (measure by re-querying after
     `el.blur()`+refocus is NOT allowed — instead snapshot style before via
     matching unfocused sibling state: implement as: capture computed style of
     the element while focused, then compare with the same element's style with
     `:focus-visible` polyfill check via `el.matches(":focus-visible")` and a
     forced `document.activeElement.blur()` at walk end replay — keep simple:
     compare focused computed style against the style captured for that element
     during census (unfocused baseline));
   - cycle detection: stop when the first element repeats or focus leaves the
     document;
   - finding `runtime/probe/focus-visible` (serious, 2.4.7 AA) when focused
     styles differ from baseline by NO perceptible indicator: no outline
     (style ≠ none && width ≥ 1px), no box-shadow delta, no border-color delta,
     no background-color delta — all four absent → violation; screenshot the
     element (evidence);
   - finding `runtime/probe/focus-order-jump` (advisory) when consecutive stops
     jump backwards in document order by more than 1 position (records the
     pair) — advisory because intent can be legitimate.
3. keyboard-reach probe: census set minus Tab-visited set → each unreached
   element yields `runtime/probe/keyboard-unreachable` (serious, 2.1.1 A) with
   descriptor + census screenshot. Elements reachable only via arrow-key
   composite widgets (role=tab/menuitem/radio inside a container with
   `aria-activedescendant` or roving tabindex — container itself visited) are
   excluded (documented heuristic: if an unreached element's composite
   container was visited, skip with debug note).
4. Both probes write raw JSON records (walk sequence, census, exclusions) to the
   EvidenceSink (issue 22 interface).
5. Timeouts: whole probe pair ≤ 30 s per page (constant); exceeding → probe
   aborted, advisory finding `runtime/probe/probe-timeout`.

## Acceptance Criteria

- [ ] Fixture pages: (a) visible-focus page → no focus-visible findings;
      (b) `outline: none` page → finding with element screenshot;
      (c) `tabindex=-1` button → keyboard-unreachable; (d) roving-tabindex
      tablist → no false positive (U4 tuning case).
- [ ] Cycle detection terminates on focus-trap fixture; walk data in evidence.
- [ ] Census exclusion rules unit-tested against a synthetic DOM page.
- [ ] Determinism: repeated runs identical fingerprints.
- [ ] U4 note: false-positive tuning results recorded in code comments +
      DESIGN §2.3 row updated in the same PR.

## Validation

```bash
npm test -- src/audit/probes
```

## Dependencies

22.

## Non-goals

Reflow/target-size/structure (24); screen-reader simulation; arrow-key widget
traversal.

## Design References

DESIGN.md §11.4, §2.3 U4, §7 (finding shape); research/2026-07-runtime-audit-tooling.md.
