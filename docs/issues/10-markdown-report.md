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

1. `MarkdownReportInput` — this issue OWNS the full input contract (issue 13's
   `FixResultSummary` must be structurally assignable to `fixResults`;
   integration is verified in issue 17):
   ```ts
   {
     kind: "scan" | "audit" | "fix";
     run: { runId: string; toolVersion: string; startedAt: string;
            durationMs: number; command?: string };   // command = argv line for
                                                      // the reproduction footer,
                                                      // supplied by run-context
     findings: Finding[];               // baselineStatus/confidence read off the
                                        // Finding fields (DESIGN §7.1)
     fixResults?: {
       entries: Array<{ fingerprint: string; ruleId: string; file: string;
         status: "fixed" | "fix_skipped" | "fix_deferred" | "fix_failed"
               | "needs_human";
         className: "auto_safe" | "auto_review" | "content_required";
         llmGenerated?: true; note?: string; reason?: string }>;
     };
     baselineActive: boolean;           // adds new/known columns when true
     staleBaselineCount?: number;
   }
   ```
2. Layout (order fixed, deterministic output — no timestamps beyond run meta):
   - H2 title `a11y-bot <kind> report`, one-line stats sentence.
   - "Summary" table: severity × count; when `baselineActive`, split columns
     new/known computed from each finding's `baselineStatus`.
   - "Top rules" table: ruleId, count, WCAG SCs, fixability (top 10 by count;
     ties broken by ruleId lexicographic).
   - Per-file (static) / per-target (runtime) H3 sections with finding tables:
     location, severity, ruleId, message (message truncated 120 chars).
     Ordering (deterministic): files/targets lexicographic by path/name;
     findings within a section by (line, column, ruleId) for static and by
     (selector, ruleId) for runtime.
   - `advisory` (ux) findings under a separate H3 "Advisory UX findings
     (AI-assisted)" with a confidence column read from `Finding.confidence`
     (blank when absent) — clearly labeled non-gating.
   - When `fixResults` present: "Fixed" / "Needs human" / "Failed or deferred"
     tables; AI-generated fixes listed under an "AI-generated (review required)"
     H3 (DESIGN §10.2.5).
   - Footer: reproduction command line (`run.command`, omitted when absent) +
     docs link + marker slot (caller may append the PR marker comment —
     renderer itself does NOT include it).
3. Escaping: all user-content strings (messages, snippets, selectors, paths)
   are Markdown-escaped (pipes, backticks, angle brackets → entities) so hostile
   file content cannot inject Markdown/HTML into PR bodies (security: DESIGN
   §14.2 T1 surface).
4. Size control: when total findings > 300, per-file/per-target detail sections
   are replaced by ONE table `path/target × counts-by-severity` (summary and
   top-rules tables unchanged); independent of that, output is hard-capped at
   60 KiB by truncating detail sections from the end with a truncation notice
   (summary tables always survive) — keeps PR bodies under GitHub's
   65,536-char limit.
5. Pure function, no I/O; snapshot-tested. `run.command` is caller-provided
   display text and passes through the same Markdown escaping as user content.

## Acceptance Criteria

- [ ] Snapshot tests for scan-only, scan+baseline, fix, and audit inputs
      (including advisory section) — stable across runs.
- [ ] Escaping tests cover EVERY user-content surface: message, snippet,
      selector, file path, note/reason, and `run.command` — each with a
      `| <img onerror> \`` payload rendering inertly.
- [ ] Secret-leak guard (DESIGN §14.2 T3): with token-shaped strings planted in
      message/snippet inputs, the renderer output contains them only as
      escaped inert text and a test asserts env-canary values never appear
      (renderer receives already-redacted inputs; test documents the boundary).
- [ ] >300-findings collapse table + 60 KiB cap tests (1,000 synthetic
      findings); summary tables intact in both.
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
