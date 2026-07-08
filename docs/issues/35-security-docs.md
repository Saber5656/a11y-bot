# Title

SECURITY.md, threat-model doc, permissions guide

## Summary

Publish the user-facing and contributor-facing security documentation derived
from DESIGN.md §14, making the security posture auditable before OSS release.

## Context

The repository is public-first OSS. DESIGN.md §14 holds the internal model;
this issue turns it into: a disclosure policy, an operator guide (what the bot
can touch and why), and a verification checklist that release (36) gates on.

## Scope

- `SECURITY.md` (repo root), `docs/security/threat-model.md`,
  `docs/security/permissions.md`; cross-link pass over README and guides.

## Detailed Requirements

1. `SECURITY.md` (GitHub-recognized): supported-versions table (1.x),
   private disclosure channel (GitHub private vulnerability reporting enabled —
   instruction for maintainer + fallback email placeholder for the owner to
   fill; agents must not invent contact data), response-time expectation
   (triage ≤ 14 days), scope (what counts: the CLI, the Action, templates),
   out-of-scope list (user misconfiguration of their own workflows, upstream
   plugin false negatives).
2. `docs/security/threat-model.md`: DESIGN §14.1–14.3 expanded to prose with
   the T1–T12 table, each row linking to the implementing issue/module and its
   tests (traceability matrix: threat → mitigation → test file path). This doc
   is the release-audit artifact.
3. `docs/security/permissions.md` (operator guide):
   - exact GitHub token permissions per mode (check: `contents: read`; fix:
     `contents: write` + `pull-requests: write`; audit: `contents: read`),
     fine-grained PAT equivalents;
   - what the bot writes (bot branch allowlist, PR, labels) and provably never
     does (push non-bot refs, merge, delete, comment on foreign PRs);
   - LLM data-flow disclosure: exactly what leaves the machine when LLM is on
     (framed snippets, downscaled local images, evidence excerpts; never
     tokens/env/full files) and that nothing leaves when off;
   - evidence-bundle handling guidance (artifact retention, sensitive
     screenshots, `containsSensitiveInput` flag).
4. Verification checklist (embedded in threat-model doc, checked in 36's
   release gate): each T-row's test exists and passed in the latest CI run;
   `grep pull_request_target` clean; redaction tests green; dist staleness
   check green; lockfile audit (`npm audit --omit dev` policy: no high/critical
   unpatched — exceptions documented inline).
5. README security section: 5-line summary linking the three docs.

## Acceptance Criteria

- [ ] All three documents merged; T1–T12 traceability rows complete with real
      test paths (no TODO rows).
- [ ] Owner-provided disclosure contact confirmed with the human maintainer
      before merge (explicit sign-off recorded in the PR).
- [ ] Permissions doc matches templates (32) exactly — cross-checked by a doc
      test extracting permission blocks from templates and diffing against the
      doc's table.
- [ ] Release checklist section consumed by issue 36 (link stable).

## Validation

```bash
npm test -- docs-consistency   # template/permissions extraction test
```

## Dependencies

30, 32.

## Non-goals

Implementing new mitigations (they live in their issues); formal audit; bug
bounty setup.

## Design References

DESIGN.md §14 (all); ISSUE_PLAN §Validation.7.
