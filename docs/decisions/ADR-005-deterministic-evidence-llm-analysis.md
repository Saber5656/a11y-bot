# ADR-005: Runtime UX auditing = deterministic evidence collection + post-hoc LLM analysis

- Status: accepted (2026-07-08, approved by product owner)

## Context

The owner wants auditing beyond static markup ("a11y includes UX"), suggesting
browser automation possibly up to computer-use-style agents. Options: (a)
deterministic probes only, (b) deterministic probes + LLM analysis of collected
evidence, (c) autonomous LLM browser agent in the loop.

## Decision

Option **(b)** for v1:

1. A scripted Playwright session (chromium) collects an **evidence bundle**:
   axe results, screenshots per viewport, focus-order walk, keyboard-reachability
   census, reflow/target-size measurements, structure outline, and user-defined
   scripted flows from config.
2. After the browser closes, an optional LLM pass reads the bundle and emits
   schema-constrained, advisory UX findings (skipped without a key).
3. The LLM never controls the browser in v1.

## Rationale

- Reproducibility: identical page → identical evidence; only the advisory layer is
  model-dependent, and it cannot fail CI.
- Cost/wall-clock bounded by probe design, not by agent exploration.
- No action-safety sandbox needed (no destructive clicks possible by design).

## Consequences

- "Can a user complete flow X with only a keyboard?" is approximated by
  user-authored scripted flows + reachability probes, not open-ended agent trials.
- An autonomous agent mode is a v2 candidate that must write into the same evidence
  bundle format (the abstraction boundary defined in DESIGN.md §11.6).
