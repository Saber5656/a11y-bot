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
   - the audited page per target (v1): the target's base URL exactly — the
     configured `url` as-is, or `http://127.0.0.1:<port>/` for
     staticDir/command variants (additional pages only via flow `goto`, 25);
   - navigation: `page.goto(pageUrl, { waitUntil: "load", timeout:
     config.audit.pageTimeoutMs })` + 500 ms settle delay constant; per-page
     failure → finding `runtime/probe/page-load-failed` (operational rule per
     DESIGN §11.4 — REGISTERED BY THIS ISSUE with catalog message) and skip
     remaining work for that page.
2. axe scan:
   - `new AxeBuilder({ page }).withTags(["wcag2a","wcag2aa","wcag21a",
     "wcag21aa","wcag22a","wcag22aa"])` (constant, documented; a tag with no
     rules in the installed axe version is harmless);
   - each violation node → one Finding: ruleId `runtime/axe/<axe-id>`,
     severity from impact (§7.4), `target = { name, url, selector:
     node.target.join(" "), viewport }`, wcag refs parsed from axe tags
     (`wcag111` → `1.1.1`; level/version inferred from tag; unparsable tags
     ignored with debug log), fixability `none`, `evidenceRefs` = element
     screenshot path when capture succeeds (see 4);
   - axe `incomplete` results → findings `runtime/axe/<id>` with
     `advisory: true` (allowed on runtime findings per DESIGN §7.1) and note
     "needs review"; emitted only when config `audit.includeIncomplete: true`
     (key exists in the schema from issue 02).
3. Fingerprint inputs use the §7.3 runtime form (target name + normalized
   selector) — verify nth-child stripping when element has id/data-testid.
4. Element screenshot capture: `locator.screenshot()` with 2 s timeout, stored
   via the evidence API. **This issue defines the `EvidenceSink` interface**
   (26 implements it; an in-memory stub here keeps 22 testable standalone):
   ```ts
   interface EvidenceSink {
     // rel is bundle-relative under targets/<target>/<viewport>/,
     // validated ^[a-z0-9/._-]+\.(json|png)$ — invalid → InternalError (T11)
     writeJson(rel: string, data: unknown): Promise<string>;   // returns ref
     writeScreenshot(rel: string, png: Buffer): Promise<string | null>;
                                    // null once audit.maxScreenshots reached
     markFlowSensitive(flow: string): void;  // sets containsSensitiveInput
   }
   ```
   The sink contract INCLUDES scrubbing: implementations must apply
   `audit.scrubParams` masking to every string field of `writeJson` payloads
   (the stub mimics this so tests exercise scrubbed output).
5. Determinism: two consecutive runs against the fixture site produce identical
   finding fingerprint sets (test tolerance: zero drift).

## Acceptance Criteria

- [ ] Fixture site (intentional violations: missing alt, low contrast, missing
      label) yields expected `runtime/axe/*` findings across 2 viewports.
- [ ] Impact→severity, tag→WCAG parsing, selector normalization unit-tested.
- [ ] Missing-browser EnvError path tested (env override forces launch failure).
- [ ] Determinism double-run test green.
- [ ] `includeIncomplete` toggle behavior tested (off by default; advisory
      findings never gate).
- [ ] Security (DESIGN §14.2 T7/T11): console/request-URL evidence written
      through the sink shows `?token=***`-style scrubbing (stub assertion);
      an invalid sink path (`../x.json`) → InternalError.
- [ ] Operational rule `runtime/probe/page-load-failed` registered with
      catalog message (unreachable-page fixture test).

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
