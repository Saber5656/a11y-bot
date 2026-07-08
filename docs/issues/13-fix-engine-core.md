# Title

Fix engine core: plan/apply/verify state machine

## Summary

Implement the fixer-agnostic engine: fixability policy, text-edit patch model,
per-file apply pipeline with rollback, and the re-lint verification loop.

## Context

DESIGN.md §9.1–9.2 is normative, including the policy guardrails (never insert
empty alt, never delete content nodes, stay in range, no active content). Fixers
(14–16, 20) plug into this engine via a registration interface.

## Scope

- `src/fix/engine.ts`, `src/fix/patch.ts`, `src/fix/verify.ts`,
  `src/fix/catalog.ts` (fixer registry).

## Detailed Requirements

1. Fixer interface:
   ```ts
   interface Fixer {
     ruleId: string;                       // unified id it repairs
     class: "auto_safe" | "auto_review" | "content_required";
     plan(finding: Finding, file: SourceFile): Promise<FixPlan | null>;
   }
   interface FixPlan {
     findingFingerprint: string;
     edits: TextEdit[];                    // { startOffset, endOffset, newText }
     note?: string;                        // shown in PR body
     llmGenerated?: true;
   }
   ```
   `plan` returning null = "cannot fix this instance" (reported `fix_skipped`).
2. Policy enforcement in the ENGINE (not trusted to fixers):
   - every edit within finding range ± the same element's attribute span
     (fixer supplies `allowedSpan`; engine validates edits ⊆ allowedSpan);
   - reject `newText` containing `<script`, `javascript:`, `data:`,
     `on[a-z]+=` patterns, or control chars (shared validator with issue 19);
   - reject class not enabled in `fix.classes` / `fix.llm` config;
   - reject empty-alt insertion unless plan is flagged
     `decorativeConfirmed: true` (only the LLM alt fixer with `auto_review` may
     set it — DESIGN §10.3).
3. Apply pipeline per file (state machine DISCOVERED→…→COMMITTED per DESIGN
   §9.2): sort plans by startOffset desc; overlap → keep first, defer second
   (`fix_deferred`); apply to in-memory copy.
4. Verify: re-parse + re-lint the modified file with the same profile;
   pass iff (a) original finding's fingerprint absent, (b) no findings present
   that were absent before (fingerprint set comparison). Fail → rollback file,
   result `fix_failed` with reason enum
   (`verify_finding_persists | verify_new_findings | parse_error`).
5. Results model `FixResultSummary` (consumed by markdown/PR/report):
   per finding: `fixed | fix_skipped | fix_deferred | fix_failed | needs_human`
   (+ class, llmGenerated, note, reason).
6. Idempotence: running the engine twice on the same tree yields zero edits the
   second time (verified in tests via double-run).
7. Determinism: no reordering of unrelated file content; unchanged files remain
   byte-identical; changed files preserve original newline style (LF/CRLF
   detection) and BOM.

## Acceptance Criteria

- [ ] State machine transitions and rollback covered by unit tests (mock fixer).
- [ ] Policy tests: out-of-span edit rejected; `on*=`/script payload rejected;
      disabled class rejected; empty-alt guard enforced.
- [ ] Verify loop: fixture where a naive fix introduces a new violation →
      rollback + `verify_new_findings`.
- [ ] Overlap deferral and double-run idempotence tests.
- [ ] CRLF/BOM preservation tests.

## Validation

```bash
npm test -- src/fix
```

## Dependencies

09 (re-lint uses scan profiles).

## Non-goals

Concrete fixers (14–16, 20), CLI (17), git/PR (29/30).

## Design References

DESIGN.md §9.1–9.2, §10.2 (shared validators), §16 (golden methodology).
