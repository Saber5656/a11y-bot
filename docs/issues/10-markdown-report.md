# Title

Markdown report renderer (shared by PR body / step summary)

## Summary

Implement the Markdown reporter used for `--format markdown`, the fix-PR body
(issue 30), and `GITHUB_STEP_SUMMARY` (issue 31).

## Context

One renderer, three consumers — so it takes structured input (findings + run
meta + optional fix results + optional audit meta) and renders deterministic
GitHub-flavored Markdown. DESIGN.md §12.1.

## Scope

- `src/report/markdown.ts` with entry
  `renderMarkdownReport(input: MarkdownReportInput): string`.

## Detailed Requirements

1. `MarkdownReportInput`:
   ```ts
   {
     kind: "scan" | "audit" | "fix";
     run: RunMeta;                       // runId, version, startedAt, durationMs
     findings: Finding[];
     fixResults?: FixResultSummary;      // defined in issue 13; optional pre-13:
                                         // type imported lazily, field optional
     baseline?: { newCount: number; knownCount: number; staleCount: number };
   }
   ```
2. Layout (order fixed, deterministic output — no timestamps beyond run meta):
   - H2 title `a11y-bot <kind> report`, one-line stats sentence.
   - "Summary" table: severity × count (+ "new vs baseline" column when
     `baseline` present).
   - "Top rules" table: ruleId, count, WCAG SCs, fixability (top 10 by count).
   - Per-file (static) / per-target (runtime) H3 sections with finding tables:
     location, severity, ruleId, message (message truncated 120 chars).
   - `advisory` (ux) findings under a separate H3 "Advisory UX findings
     (AI-assisted)" with confidence column — clearly labeled non-gating.
   - When `fixResults` present: "Fixed" / "Needs human" / "Failed or deferred"
     tables; AI-generated fixes listed under an "AI-generated (review required)"
     H3 (DESIGN §10.2.5).
   - Footer: reproduction command line + docs link + marker slot (caller may
     append the PR marker comment — renderer itself does NOT include it).
3. Escaping: all user-content strings (messages, snippets, selectors, paths)
   are Markdown-escaped (pipes, backticks, angle brackets → entities) so hostile
   file content cannot inject Markdown/HTML into PR bodies (security: DESIGN
   §14.2 T1 surface).
4. Size control: > 300 findings → per-file sections collapse to counts with a
   note; hard cap output at 60 KiB (truncation notice with counts preserved) —
   keeps PR bodies under GitHub's 65,536-char limit.
5. Pure function, no I/O; snapshot-tested.

## Acceptance Criteria

- [ ] Snapshot tests for scan-only, scan+baseline, fix, and audit inputs
      (including advisory section) — stable across runs.
- [ ] Escaping test: finding message containing `| <img onerror> \`` renders
      inertly (no table breakage, no raw HTML).
- [ ] 60 KiB cap test with 1,000 synthetic findings; summary tables intact.
- [ ] Renderer imported by `scan --format markdown` (wire the flag now) writing
      `<outputDir>/scan.md`.

## Validation

```bash
npm test -- src/report/markdown
node dist/cli/index.js scan fixtures/static-html --format markdown --output .a11ybot/reports && head -40 .a11ybot/reports/scan.md
```

## Dependencies

03, 09.

## Non-goals

PR body assembly/marker (30), step-summary writing (31), audit orchestration (27).

## Design References

DESIGN.md §12.1, §10.2 (AI-marking), §13.2 (body reuse), §14.2 T1.
