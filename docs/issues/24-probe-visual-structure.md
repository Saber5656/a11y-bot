# Title

Probes: reflow, target size, structure outline

## Summary

Implement the remaining deterministic probes: 320-px reflow check, WCAG 2.5.8
target-size measurement, and the document structure outline (headings,
landmarks, title, lang).

## Context

DESIGN.md §11.4 rows 3–5. The structure outline doubles as evidence for the LLM
analyst (28) and as the source of several cheap findings.

## Scope

- `src/audit/probes/reflow.ts`, `src/audit/probes/target-size.ts`,
  `src/audit/probes/structure.ts`; consumes the shared census module
  `src/audit/probes/interactive-census.ts` created by issue 23.
- Registry entries + catalog messages for every `runtime/probe/*` rule this
  issue emits.

## Detailed Requirements

1. reflow probe:
   - dedicated context at viewport `{ width: 320, height: 800 }` (constant,
     runs once per target regardless of configured viewports);
   - after load+settle: violation iff
     `document.documentElement.scrollWidth > 320 + 8` (8 px tolerance constant)
     → finding `runtime/probe/reflow-horizontal-scroll` (moderate, 1.4.10 AA)
     with full-page screenshot evidence + one offending element identified as:
     among all elements with `scrollWidth > clientWidth`, the one with the
     largest `scrollWidth` (tie → first in document order), best-effort
     selector per the issue-23 selector rules.
2. target-size probe (runs on each configured viewport):
   - census from the issue-23 shared module; for each element compute
     effective size = bbox; violation iff width < 24 or height < 24 CSS px AND
     NOT (a) inline text link (display inline within a text node context),
     (b) sufficient spacing: the center-to-center distance from this target to
     EVERY other census target is ≥ 24 CSS px (deterministic approximation of
     the WCAG 2.5.8 spacing exception — the exact SC allows a 24-px-diameter
     non-overlapping circle; the approximation and its bias are documented in
     code and report docsUrl), (c) `input[type=checkbox|radio]` with an
     associated label whose combined hit area ≥ 24;
   - finding `runtime/probe/target-size` (moderate, 2.5.8 AA), grouped: one
     finding per element, capped at 25 per page (overflow noted in probe JSON).
3. structure probe:
   - extract: `document.title` (finding `runtime/probe/missing-title`, serious,
     2.4.2 if empty), `html[lang]` (`runtime/probe/missing-lang`, serious,
     3.1.1 if absent/empty or failing the primary-tag regex
     `^[a-zA-Z]{2,3}(-[a-zA-Z0-9]{2,8})*$` — no BCP-47 library in v1,
     documented),
     landmarks (`main, nav, banner, contentinfo` via tag or role — missing
     `main` → `runtime/probe/missing-main-landmark`, moderate, bestPractice),
     heading tree (tag, text ≤ 80 chars, depth) — `h1` absent →
     `runtime/probe/missing-h1` (moderate, bestPractice); level skip (h2→h4) →
     `runtime/probe/heading-skip` (minor, bestPractice);
   - full outline JSON (headings, landmarks, title, lang, meta viewport) written
     to EvidenceSink as `outline.json` (consumed by 26/28).
4. All findings deduplicate against axe where overlap exists: if the same page
   already has `runtime/axe/document-title` (etc.), the probe suppresses its
   duplicate (suppression map constant: title/lang/landmark rules ↔ axe ids
   `document-title`, `html-has-lang`, `landmark-one-main`; heading-order ↔
   `heading-order`) — axe wins (richer metadata). Unit-tested.
5. Timeout ≤ 20 s per page for the trio; abort → `runtime/probe/probe-timeout`
   (advisory operational rule registered by issue 23; reused here).
6. All evidence writes go through the EvidenceSink (issue 22) — scrubbing and
   path validation are sink-enforced (T7/T11).

## Acceptance Criteria

- [ ] Fixture with fixed-width 1000-px element → reflow finding; responsive
      fixture → none.
- [ ] Target-size fixtures: 16-px icon button → finding; 16-px inline link →
      exempt; spaced small buttons → exempt (spacing rule); cap at 25 tested
      synthetically.
- [ ] Structure fixtures: missing title/lang/h1/main and h2→h4 skip each
      produce exactly one finding; axe-suppression verified (run with axe
      enabled, no duplicates).
- [ ] outline.json schema (add to `schemas/`) validated in tests.
- [ ] Security (T7): outline.json for a fixture page whose links carry
      `?token=x` query params is written scrubbed (sink integration test).
- [ ] Registry sweep: every emitted rule id registered with catalog message.
- [ ] Determinism double-run test.

## Validation

```bash
npm test -- src/audit/probes
```

## Dependencies

22, 23 (shared interactive-census module).

## Non-goals

Zoom-via-page-zoom emulation (viewport method chosen), contrast probing (axe
covers), motion preferences.

## Design References

DESIGN.md §11.4; research/2026-07-runtime-audit-tooling.md.
