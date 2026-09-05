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
  shared `src/audit/probes/interactive-census.ts` (also consumed by issue 24).
- Registry entries + catalog messages for `runtime/probe/focus-visible`,
  `runtime/probe/focus-order-jump` (bestPractice, advisory),
  `runtime/probe/keyboard-unreachable`, and the shared operational rule
  `runtime/probe/probe-timeout` (minor, bestPractice, advisory — DESIGN §11.4
  operational findings).

## Detailed Requirements

1. Interactive census (shared): collect elements matching
   `a[href], button, input:not([type=hidden]), select, textarea, summary,
   [tabindex]:not([tabindex="-1"]), [role=button], [role=link], [role=checkbox],
   [role=radio], [role=tab], [role=menuitem], [contenteditable=true]`,
   excluding `disabled`, `aria-hidden="true"` ancestors, `visibility:hidden`/
   `display:none`, and zero-area boxes. Output: descriptor list
   `{ selector, role, name, bbox }` where `selector` preference order is
   (1) `#id` when unique, (2) `[data-testid="…"]` when unique, (3) the
   tag:nth-of-type chain from `body` (e.g.
   `body > div:nth-of-type(2) > button:nth-of-type(1)`); `name` precedence is
   `aria-label` → text of `aria-labelledby` referents → associated `<label>`
   text → own text content, trimmed and capped at 80 chars.
2. focus-order probe:
   - during census (before any focus), capture each element's **unfocused
     baseline**: computed values of `outline-style`, `outline-width`,
     `outline-color`, `box-shadow`, `border-color`, `background-color`;
   - start at `<body>`; press Tab up to `min(200, census.length * 2)` times;
     after each stop record: active element descriptor, bbox, and the same six
     computed values in the focused state;
   - **delta definition (normative)**: a property has a delta iff its focused
     computed-value string differs from the baseline string (exact string
     inequality — no numeric thresholds);
   - **violation condition** for `runtime/probe/focus-visible` (serious,
     2.4.7 AA): ALL of the following are true — (a) no visible outline in the
     focused state (`outline-style` is `none` OR `outline-width` < 1px),
     (b) no `box-shadow` delta, (c) no `border-color` delta, (d) no
     `background-color` delta. Screenshot the element as evidence.
     (`background-color` inclusion is per DESIGN §11.4 as updated.)
   - cycle detection: stop when the first element repeats or focus leaves the
     document;
   - finding `runtime/probe/focus-order-jump` (`advisory: true`,
     `bestPractice: true` — no WCAG ref) when consecutive stops jump backwards
     in document order by more than 1 position (records the pair).
3. keyboard-reach probe: census set minus Tab-visited set → each unreached
   element yields `runtime/probe/keyboard-unreachable` (serious, 2.1.1 A) with
   descriptor + census screenshot. Elements reachable only via arrow-key
   composite widgets (role=tab/menuitem/radio inside a container with
   `aria-activedescendant` or roving tabindex — container itself visited) are
   excluded (documented heuristic: if an unreached element's composite
   container was visited, skip with debug note).
4. Both probes write raw JSON records (walk sequence, census, exclusions) to
   the EvidenceSink (issue 22 interface) — scrubbing and path validation are
   sink-enforced (T7/T11).
5. Timeouts: whole probe pair ≤ 30 s per page (constant); exceeding → probe
   aborted, finding `runtime/probe/probe-timeout` (advisory, registered by
   this issue).

## Acceptance Criteria

- [ ] Fixture pages: (a) visible-focus page → no focus-visible findings;
      (b) `outline: none` page → finding with element screenshot;
      (c) `tabindex=-1` button → keyboard-unreachable; (d) roving-tabindex
      tablist → no false positive (U4 tuning case).
- [ ] Cycle detection terminates on focus-trap fixture; walk data in evidence.
- [ ] Census exclusion rules and selector/name precedence unit-tested against
      a synthetic DOM page.
- [ ] Security (T7/T11): walk JSON containing a URL with `?token=x` is written
      scrubbed (via sink); registry entries for all four rules present with
      catalog messages (sweep test).
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
