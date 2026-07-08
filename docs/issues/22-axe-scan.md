# Title

Playwright session + axe-core scan to unified findings

## Summary

Implement the browser session manager (chromium, per target×viewport contexts)
and the axe-core scan producing `runtime/axe/*` unified findings with evidence
capture hooks.

## Context

DESIGN.md §11.2–11.3; research doc fixes `@axe-core/playwright` usage and the
WCAG tag set. This issue also establishes the session plumbing that probes
(23/24) and flows (25) reuse.

## Scope

- `src/audit/session.ts` (browser/context lifecycle), `src/audit/axe.ts`.

## Detailed Requirements

1. Session:
   - `launchAudit(config)` → chromium headless (`playwright` package);
     missing browser binary → EnvError exit 4 with hint
     `npx playwright install chromium` (DESIGN §18) — detect via launch error
     message match;
   - per target×viewport: fresh `BrowserContext`
     (`viewport`, `deviceScaleFactor: 1`, `reducedMotion: "no-preference"`,
     `locale: "en-US"`, `timezoneId: "UTC"` for determinism);
   - page event taps: console messages (type error/warning), request failures →
     ring buffers exposed for the evidence bundle (26);
   - navigation: `page.goto(baseUrl + path, { waitUntil: "load", timeout:
     config.audit.pageTimeoutMs })` + 500 ms settle delay constant; per-page
     failure → target-page finding `runtime/probe/page-load-failed`
     (severity serious) and skip remaining steps for that page.
2. axe scan:
   - `new AxeBuilder({ page }).withTags(["wcag2a","wcag2aa","wcag21a",
     "wcag21aa","wcag22aa"])` (constant, documented);
   - each violation node → one Finding: ruleId `runtime/axe/<axe-id>`,
     severity from impact (§7.4), `target = { name, url, selector:
     node.target.join(" "), viewport }`, wcag refs parsed from axe tags
     (`wcag111` → `1.1.1`; level/version inferred from tag; unparsable tags
     ignored with debug log), fixability `none`, `evidenceRefs` = element
     screenshot path when capture succeeds (see 4);
   - axe `incomplete` results → advisory findings `runtime/axe/<id>` with
     `advisory: true` and note "needs review" (config off-switch
     `audit.includeIncomplete: false` default false — ADD to schema in this
     issue with default false; document in DESIGN §6.2 in same PR).
3. Fingerprint inputs use the §7.3 runtime form (target name + normalized
   selector) — verify nth-child stripping when element has id/data-testid.
4. Element screenshot capture: `locator.screenshot()` with 2 s timeout, stored
   via the evidence API (26 provides the sink; until then an in-memory stub
   interface `EvidenceSink` defined HERE and implemented in 26 — keeps 22
   testable standalone).
5. Determinism: two consecutive runs against the fixture site produce identical
   finding fingerprint sets (test tolerance: zero drift).

## Acceptance Criteria

- [ ] Fixture site (intentional violations: missing alt, low contrast, missing
      label) yields expected `runtime/axe/*` findings across 2 viewports.
- [ ] Impact→severity, tag→WCAG parsing, selector normalization unit-tested.
- [ ] Missing-browser EnvError path tested (env override forces launch failure).
- [ ] Determinism double-run test green.
- [ ] `includeIncomplete` toggle behavior tested; schema + DESIGN updated.

## Validation

```bash
npx playwright install chromium
npm test -- src/audit/axe src/audit/session
```

## Dependencies

21, 03.

## Non-goals

Probes (23/24), flows (25), bundle persistence (26), LLM analysis (28).

## Design References

DESIGN.md §11.2–11.3, §7.3–7.4, §18; research/2026-07-runtime-audit-tooling.md.
