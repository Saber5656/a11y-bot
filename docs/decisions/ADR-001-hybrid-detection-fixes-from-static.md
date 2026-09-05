# ADR-001: Hybrid detection; automated fixes only from static findings

- Status: accepted (2026-07-08, approved by product owner)
- Deciders: product owner (takagiyasushi), Fable (design)

## Context

a11y-bot's mission is "detect accessibility violations and open fix PRs". Detection
can happen (a) statically on repository sources, or (b) at runtime in a real
browser. Static findings carry exact source positions; runtime findings reference
rendered DOM nodes, whose reverse-mapping to source files (across JSX/Vue compile
steps) is unreliable. The owner also requires UX-level auditing ("a11y is not only
static UI"), which only runtime evidence can support.

## Decision

Run **both** engines with a strict division of authority:

| Engine | Output | May produce automated patches? |
|---|---|---|
| Static scan (ESLint-based) | Findings with file/range | **Yes** — deterministic codemods + optional LLM content fixes |
| Runtime audit (Playwright + axe + probes) | Findings with URL/selector + evidence bundle | **No** in v1 — report and CI gate only |
| LLM UX analysis (over evidence) | Advisory findings | No, and never gates CI |

## Consequences

- Fix PRs are always source-grounded → low false-fix risk, verifiable by re-lint.
- Runtime-only violations (contrast, focus visibility, reflow…) surface in reports
  and can gate CI (axe-derived, deterministic), guiding manual fixes.
- Runtime→source mapping research (needed for "fix from runtime finding") is
  deferred to v2 and isolated behind the unified finding model.
