# Title

Workflow templates + permissions/fork-safety documentation

## Summary

Provide copy-paste GitHub workflow templates for the three operation modes with
exact least-privilege permission blocks and explicit fork-PR safety rules.

## Context

DESIGN.md §13.3 + threat rows T4/T5. Templates are the primary adoption surface
and the place where security posture is either preserved or lost — they are a
deliverable with acceptance criteria, not ad-hoc README snippets.

## Scope

- `examples/workflows/pr-check.yml`, `examples/workflows/scheduled-fix.yml`,
  `examples/workflows/audit.yml`; docs page `docs/guides/workflows.md`
  (user-facing usage guide).

## Detailed Requirements

1. `pr-check.yml`:
   - trigger `pull_request` (NOT `pull_request_target`);
   - `permissions: { contents: read }` at workflow level;
   - steps: checkout → `uses: <owner>/a11y-bot@v1` `mode: check` — plus a
     commented optional SARIF upload block (`github/codeql-action/upload-sarif`,
     needs `security-events: write`, called out in comment).
2. `scheduled-fix.yml`:
   - triggers `schedule` (weekly cron example) + `workflow_dispatch`;
   - `permissions: { contents: write, pull-requests: write }`;
   - condition `if: github.repository == '<owner>/<repo>'` template comment
     (avoid forks running crons);
   - steps: checkout (fetch-depth 0 comment on why not needed → depth 1 fine;
     document) → action `mode: fix` with `github-token:
     ${{ secrets.GITHUB_TOKEN }}`; commented variant using a PAT for
     workflows-triggering-workflows caveat (bot PRs from GITHUB_TOKEN don't
     trigger CI — documented tradeoff + recommended
     `pull_request` re-run tip).
3. `audit.yml`:
   - trigger `workflow_dispatch` + optional cron;
   - `permissions: { contents: read }`;
   - steps: checkout → `npx playwright install --with-deps chromium` → build
     the user's site (placeholder comment) → action `mode: audit` →
     `actions/upload-artifact` of the evidence dir with a comment warning about
     screenshots of authenticated/sensitive pages (T7) and the
     `containsSensitiveInput` manifest flag.
4. Every template: pinned action versions (`@v4` etc. current at
   implementation), no `pull_request_target` anywhere, `concurrency` group to
   prevent overlapping runs, timeout-minutes set (15/30/30).
5. `docs/guides/workflows.md`: table of the three templates (purpose,
   permissions, secrets exposure), fork-PR rules in a highlighted section
   ("Never run fix or LLM modes on pull_request_target; check mode is safe on
   pull_request because it needs no secrets"), token permission matrix
   (mirrors DESIGN §13.2 hints), and LLM key setup
   (`A11YBOT_LLM_API_KEY` as Actions secret; never echo).
6. Templates are lint-validated in CI with `action-validator` (or yaml-schema
   check if the tool is unmaintained — implementer verifies and records
   choice).

## Acceptance Criteria

- [ ] Three templates present, YAML-valid in CI, versions pinned, permission
      blocks exactly as specified.
- [ ] `grep -r pull_request_target examples/ docs/guides` → no hits (CI check).
- [ ] Guide covers: permissions matrix, fork rules, LLM secret setup, SARIF
      option, artifact warning — reviewed against DESIGN §14.2 T3/T4/T5/T7
      checklist (checklist included in PR description).
- [ ] scheduled-fix template executed successfully on the scratch repo used in
      issue 30's manual validation (evidence linked).

## Validation

```bash
npm test -- workflows   # yaml validation test
grep -r pull_request_target examples/ docs/guides && exit 1 || true
```

## Dependencies

31.

## Non-goals

Repository rulesets/branch-protection automation (user's own hardening);
reusable workflows (`workflow_call`, v2).

## Design References

DESIGN.md §13.3, §14.2 T4/T5/T7; ADR-006.
