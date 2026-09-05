# Research: Conformance standards — WCAG 2.2 / ISO / JIS X 8341-3 (2026-07)

Status: verified 2026-07-08 (WAIC announcements, industry blogs).
Decision consumer: DESIGN.md §3, rule registry WCAG mapping.

## Facts

- WCAG 2.2 is the current W3C Recommendation; success criterion **4.1.1 Parsing was
  removed** in 2.2.
- WCAG 2.2 was adopted as **ISO/IEC 40500:2025**.
- Japan: JIS X 8341-3 is being revised to be an identical standard to
  ISO/IEC 40500:2025 (= WCAG 2.2). WAIC formed the revision drafting committee on
  2025-10-30, targeting a completed draft around **May 2026**; Japan-specific
  content moves to informative annexes. Until publication, JIS X 8341-3:2016
  (aligned with WCAG 2.0) remains current.
- Practical implication reported by WAIC seminar material: AA conformance will
  require ~1.4× the success criteria count relative to the 2016 edition.

Sources: waic.jp news 2025-11-13 (committee formation), WAIC seminar PDF 2026-02-06
(revision outline), mitsue.co.jp a11y blog 2025-08 (ISO/IEC 40500:2025 timing).

## Decisions derived

1. **Target: WCAG 2.2 Level AA by default.** Configurable level (`A`, `AA`) and
   version tagging per rule so reports can filter.
2. Rule registry stores `wcag` refs as SC numbers with version metadata
   (e.g., `2.5.8` is 2.2-only). No rule maps to 4.1.1.
3. Ship a **JIS X 8341-3 correspondence note** in docs: WCAG 2.2 AA conformance
   positions users for the upcoming revised JIS (identical standard); the 2016
   edition maps via WCAG 2.0 subset.
4. Report language is English (message catalog keyed by rule id, so a `ja` catalog
   can be added in v2 without code changes).
