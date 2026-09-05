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
   interface SourceFile {
     path: string;            // repo-relative POSIX
     text: string;            // original content
     newline: "lf" | "crlf";  // detected; preserved on write
     bom: boolean;            // preserved on write
   }
   interface Fixer {
     ruleId: string;                       // unified id it repairs
     class: "auto_safe" | "auto_review" | "content_required";
     plan(finding: Finding, file: SourceFile): Promise<FixPlan | null>;
   }
   interface FixPlan {
     findingFingerprint: string;
     edits: TextEdit[];                    // { startOffset, endOffset, newText }
     allowedSpan: { startOffset: number; endOffset: number };
                                           // the owning element's tag span; the
                                           // engine validates edits ⊆ allowedSpan
     note?: string;                        // shown in PR body
     llmGenerated?: true;
     decorativeConfirmed?: true;           // LLM alt fixer only (DESIGN §10.3)
   }
   ```
   `plan` returning null = "fixer ran but cannot fix this instance" (reported
   `fix_skipped`). Findings whose registry fixability is `manual`, or
   `content_required` while LLM fixes are unavailable, never invoke a fixer and
   are reported `needs_human` (this is the needs_human/fix_skipped boundary).
2. Policy enforcement in the ENGINE (not trusted to fixers), via
   `src/fix/patch-validators.ts` — this module is the SINGLE source of the
   active-content rejection list; issue 19 imports and extends it for LLM
   surfaces:
   - every edit within `allowedSpan` (violation → plan rejected, InternalError
     counter incremented, finding reported `fix_failed` reason `policy`);
   - reject `newText` containing (case-insensitive): `<script`, `<style`,
     `<iframe`, `<object`, `<embed`, any HTML tag when the edit target is an
     attribute value, `javascript:`, `vbscript:`, `data:`, absolute URLs
     (`https?://`) or protocol-relative `//` in new attribute values,
     event-handler pattern `on[a-z]+\s*=`, control chars;
   - reject when the plan's class is not enabled: `auto_safe`/`auto_review`
     must be in `fix.classes`; `content_required` additionally requires
     `fix.llm: true` (or `--llm`) AND an available LLM client (key present) —
     DESIGN §9.1 table;
   - reject empty-alt insertion unless the plan carries
     `decorativeConfirmed: true` (only the LLM alt fixer may set it, and such
     plans are treated as class `auto_review` — DESIGN §9.1/§10.3).
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
- [ ] Policy tests: out-of-span edit rejected; every rejection-list pattern
      covered incl. `onclick =` (whitespace), `<style`, URL and
      protocol-relative payloads; disabled class rejected; content_required
      without key rejected; empty-alt guard enforced.
- [ ] Verify loop: fixture where a naive fix introduces a new violation →
      rollback + `verify_new_findings`.
- [ ] Overlap deferral and double-run idempotence tests.
- [ ] CRLF/BOM preservation tests.

## Validation

```bash
npm test -- src/fix
```

## Dependencies

09 (re-lint uses scan profiles). No dependency on the LLM layer — the
guardrails issue later imports this issue's `patch-validators.ts`, not the
reverse.

## Non-goals

Concrete fixers (14–16, 20), CLI (17), git/PR (29/30), LLM-side prose
validators (19).

## Design References

DESIGN.md §9.1–9.2, §10.2 (validator single-source), §14.2 T2, §16 (golden
methodology).
