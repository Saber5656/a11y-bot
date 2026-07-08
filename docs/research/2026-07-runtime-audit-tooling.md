# Research: Runtime audit tooling (2026-07)

Status: verified 2026-07-08 against npm registry.
Decision consumer: [ADR-005](../decisions/ADR-005-deterministic-evidence-llm-analysis.md), DESIGN.md §11.

## Question

What does a11y-bot use to drive a real browser, run axe checks, and collect UX
evidence deterministically in CI?

## Verified facts (npm registry, 2026-07-08)

| Package | Version | License | Notes |
|---|---|---|---|
| `playwright` | 1.61.1 | Apache-2.0 | Bundled browser download; headless Chromium in CI is first-class |
| `axe-core` | 4.12.1 | MPL-2.0 | The de-facto runtime a11y rule engine |
| `@axe-core/playwright` | 4.12.1 | MPL-2.0 | Official Playwright integration (`AxeBuilder`) |

License note: axe-core is MPL-2.0 — file-level copyleft, compatible with consuming
from an MIT project as a dependency (no modification of axe source planned).

## Engine choice

**Playwright (chromium only in v1)** over Puppeteer/Selenium:

- Deterministic auto-waiting APIs reduce flaky CI evidence.
- `page.addInitScript` / CDP access available if later probes need it.
- Official axe integration (`@axe-core/playwright`) removes injection boilerplate.
- Single-browser policy (chromium) keeps CI download/cost small; multi-browser is v2.

## Deterministic probes — feasibility notes

| Probe | Mechanism | WCAG target |
|---|---|---|
| axe scan | `AxeBuilder.analyze()` per target/viewport | broad (auto-mapped by axe) |
| focus-order walk | Repeated `keyboard.press('Tab')`, record `document.activeElement` descriptor + bounding box + computed outline/box-shadow | 2.4.3, 2.4.7 |
| keyboard reachability | Interactive-element census (selector list) vs Tab-reachable set diff | 2.1.1 |
| reflow / zoom | Re-run at 320 CSS px viewport width; detect horizontal scroll of root | 1.4.10 |
| target size | Bounding boxes of interactive elements < 24×24 CSS px (excluding inline text links) | 2.5.8 (AA, WCAG 2.2) |
| structure snapshot | Headings/landmarks/title/lang outline extraction | 1.3.1, 2.4.1, 2.4.6, 3.1.1 |

All probes are plain scripted Playwright — no LLM in the loop, reproducible, and
each emits machine-readable JSON plus screenshots into the evidence bundle.

## Autonomous browser agents (computer-use style)

Considered for "UX included" auditing. Rejected for v1 (see ADR-005): cost,
non-determinism (same page → different verdicts run-to-run), wall-clock in CI, and
the need for an action-safety sandbox (destructive-click prevention). The v1
substitute is: deterministic evidence collection (+ user-defined scripted flows)
analyzed by an LLM *after* the browser session is closed. An agentic mode remains a
v2 candidate behind the same evidence-bundle abstraction.

## SARIF

GitHub code scanning ingests SARIF 2.1.0. Practical limits to design around
(validated at implementation time): results per run upload caps, `level` mapping
(error/warning/note), and `partialFingerprints` for dedup. a11y-bot maps
severity `critical|serious → error`, `moderate → warning`, `minor → note`, and
reuses its own finding fingerprint as `partialFingerprints.a11ybotFingerprint/v1`.
