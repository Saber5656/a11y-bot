# Wave 9 concrete review-resolution addendum

- Repository: `Saber5656/a11y-bot`
- Pull request: #1
- Current PR head identity: the authoritative value is supplied by the immutable review manifest; this file intentionally omits the mutable commit SHA to avoid a self-referential identity.
- Current PR base pinned for this review: `ff2e076dc7224b7224e039a1dfc87b1deabc0484`
- This document records documentation-level handling for documentation-only PR findings. It is not a claim that implementation, runtime tests, build, CI, security, or release validation is complete.
- The immutable review manifest pins current head, base, and artifact blob identity; any later change invalidates this evidence and requires a fresh review.
- No PR review bot is triggered or rerun.

## Blocking specialist handoffs

| Role | Required evidence |
|---|---|
| QA/full validation | `tech-qa`/`tech-tester` must execute scan, report, fixer, LLM fallback, evidence, workflow, action, docs-lint, and repository-full-validation gates for this head/base. |
| Security/privacy | `tech-security`/`tech-devopssec` must accept child environment, evidence upload, token redaction, checkout credential, ref, and release boundaries. |

Missing, pending, failed, skipped, cancelled, timed-out, stale, or non-accepting specialist/full-validation evidence blocks thread resolution and merge.

## Thread contracts

### 1. Thread `PRRT_kwDOTNkC_s6PGKfA` — Keep the json-only stdout exception aligned with config outputDir.

- File: `docs/issues/09-scan-command.md`
- Line: 34
- Finding basis: Existing review finding “Keep the json-only stdout exception aligned with config outputDir.” at `docs/issues/09-scan-command.md:34`.

**Normative resolution**: At `docs/issues/09-scan-command.md:34`, JSON SHALL go to stdout only when --format json is the sole format, --output is absent, and report.outputDir is unset; otherwise it SHALL be written under the configured output directory.

**Focused verification gate**: Run JSON-only scans with no output directory, configured report.outputDir, explicit --output, and multiple formats; assert all routing cases.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 2. Thread `PRRT_kwDOTNkC_s6PGKfF` — Add staleCount to the rendered summary.

- File: `docs/issues/10-markdown-report.md`
- Line: 49
- Finding basis: Existing review finding “Add staleCount to the rendered summary.” at `docs/issues/10-markdown-report.md:49`.

**Normative resolution**: At `docs/issues/10-markdown-report.md:49`, The Markdown summary SHALL include staleCount computed from findings with baselineStatus stale whenever baseline reporting is enabled.

**Focused verification gate**: Render a baseline report containing stale findings and assert staleCount equals the stale subset.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 3. Thread `PRRT_kwDOTNkC_s6PGKfJ` — Keep the footer out of the truncation path.

- File: `docs/issues/10-markdown-report.md`
- Line: 75
- Finding basis: Existing review finding “Keep the footer out of the truncation path.” at `docs/issues/10-markdown-report.md:75`.

**Normative resolution**: At `docs/issues/10-markdown-report.md:75`, Report truncation SHALL apply only to detail sections and SHALL reserve the complete footer containing the reproduction command, documentation link, and marker slot.

**Focused verification gate**: Generate a report over the cap and assert footer markers, command, and docs link remain complete.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 4. Thread `PRRT_kwDOTNkC_s6PGKfN` — Avoid relying on a single dirty-worktree snapshot.

- File: `docs/issues/17-fix-command.md`
- Line: 58
- Finding basis: Existing review finding “Avoid relying on a single dirty-worktree snapshot.” at `docs/issues/17-fix-command.md:58`.

**Normative resolution**: At `docs/issues/17-fix-command.md:58`, The fix flow SHALL capture dirty-worktree state before candidate selection and recheck every candidate path immediately before applying a fix; any change SHALL abort the run.

**Focused verification gate**: Modify one candidate between snapshots and assert the fix aborts without writing any candidate.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 5. Thread `PRRT_kwDOTNkC_s6PGKfS` — Preserve schema validation in the 400 fallback.

- File: `docs/issues/18-llm-client.md`
- Line: 51
- Finding basis: Existing review finding “Preserve schema validation in the 400 fallback.” at `docs/issues/18-llm-client.md:51`.

**Normative resolution**: At `docs/issues/18-llm-client.md:51`, The 400 response fallback SHALL parse the instruction-embedded JSON and validate it against the original jsonSchema before returning it.

**Focused verification gate**: Return invalid and valid 400 fallback payloads; assert only the schema-valid payload returns.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 6. Thread `PRRT_kwDOTNkC_s6PGKfW` — Use deterministic fingerprint inputs.

- File: `docs/issues/20-llm-alt-text-fixer.md`
- Line: 64
- Finding basis: Existing review finding “Use deterministic fingerprint inputs.” at `docs/issues/20-llm-alt-text-fixer.md:64`.

**Normative resolution**: At `docs/issues/20-llm-alt-text-fixer.md:64`, Alt-text fingerprints SHALL use rule ID, file path, stable selector/range, and source-content hash; generated title text SHALL never be an input.

**Focused verification gate**: Run two model outputs with different wording for one source finding and assert identical fingerprints.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 7. Thread `PRRT_kwDOTNkC_s6PGKfY` — Strip repo secrets from command targets.

- File: `docs/issues/21-target-provisioner.md`
- Line: 49
- Finding basis: Existing review finding “Strip repo secrets from command targets.” at `docs/issues/21-target-provisioner.md:49`.

**Normative resolution**: At `docs/issues/21-target-provisioner.md:49`, Every command target SHALL receive a sanitized environment removing GITHUB_TOKEN, GH_TOKEN, OPENAI_API_KEY, and the variable named by llm.apiKeyEnv.

**Focused verification gate**: Spawn a target that prints its environment and assert all four credential sources are absent.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 8. Thread `PRRT_kwDOTNkC_s6PGKfZ` — Fix the checkbox/radio selector.

- File: `docs/issues/24-probe-visual-structure.md`
- Line: 45
- Finding basis: Existing review finding “Fix the checkbox/radio selector.” at `docs/issues/24-probe-visual-structure.md:45`.

**Normative resolution**: At `docs/issues/24-probe-visual-structure.md:45`, The visual-structure selector SHALL use separate selectors input[type=checkbox] and input[type=radio] for the spacing exemption.

**Focused verification gate**: Run spacing checks against real checkbox and radio inputs and assert each selector exemption applies.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 9. Thread `PRRT_kwDOTNkC_s6PGKfe` — Redact fill values by default in evidence.

- File: `docs/issues/25-flow-runner.md`
- Line: 67
- Finding basis: Existing review finding “Redact fill values by default in evidence.” at `docs/issues/25-flow-runner.md:67`.

**Normative resolution**: At `docs/issues/25-flow-runner.md:67`, Every fill-step value SHALL be serialized as *** in step JSON and uploaded evidence.

**Focused verification gate**: Create password-like and ordinary fill steps and assert both values are *** in JSON and evidence.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 10. Thread `PRRT_kwDOTNkC_s6PGKfl` — Expand the manifest/reader contract for json-only runs and capped arrays.

- File: `docs/issues/26-evidence-bundle.md`
- Line: 40
- Finding basis: Existing review finding “Expand the manifest/reader contract for json-only runs and capped arrays.” at `docs/issues/26-evidence-bundle.md:40`.

**Normative resolution**: At `docs/issues/26-evidence-bundle.md:40`, The evidence manifest SHALL contain evidenceMode full or json-only, and capped arrays SHALL accept the __truncated sentinel with omitted count.

**Focused verification gate**: Parse full and json-only manifests with capped arrays and assert mode and omitted counts survive.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 11. Thread `PRRT_kwDOTNkC_s6PGKfn` — Keep baseline findings in the report path.

- File: `docs/issues/27-audit-command.md`
- Line: 46
- Finding basis: Existing review finding “Keep baseline findings in the report path.” at `docs/issues/27-audit-command.md:46`.

**Normative resolution**: At `docs/issues/27-audit-command.md:46`, The audit pipeline SHALL collect and deduplicate all findings, render the full set with new/known markers, then apply baseline filtering only to the exit gate.

**Focused verification gate**: Compare report and exit-gate sets; assert reports retain new/known findings while the gate filters baseline entries.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 12. Thread `PRRT_kwDOTNkC_s6PGKfo` — Reject dot-segments in the branch guard.

- File: `docs/issues/29-git-engine.md`
- Line: 35
- Finding basis: Existing review finding “Reject dot-segments in the branch guard.” at `docs/issues/29-git-engine.md:35`.

**Normative resolution**: At `docs/issues/29-git-engine.md:35`, Branch names SHALL be validated with the Git ref-name validator and SHALL reject every dot or dot-dot path segment before a ref is written.

**Focused verification gate**: Attempt branch names with dot and dot-dot segments and require rejection before ref mutation.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 13. Thread `PRRT_kwDOTNkC_s6PGKf4` — Preserve the remote host when pushing.

- File: `docs/issues/29-git-engine.md`
- Line: 59
- Finding basis: Existing review finding “Preserve the remote host when pushing.” at `docs/issues/29-git-engine.md:59`.

**Normative resolution**: At `docs/issues/29-git-engine.md:59`, Pushes SHALL preserve the scheme, host, owner, and repository path parsed from the configured remote URL; github.com SHALL not be substituted.

**Focused verification gate**: Push to a non-GitHub fixture remote and assert the original scheme, host, and path are used.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 14. Thread `PRRT_kwDOTNkC_s6PGKf8` — Add issues: write to the label permission matrix.

- File: `docs/issues/30-pr-engine.md`
- Line: 48
- Finding basis: Existing review finding “Add issues: write to the label permission matrix.” at `docs/issues/30-pr-engine.md:48`.

**Normative resolution**: At `docs/issues/30-pr-engine.md:48`, The permission matrix SHALL grant issues: write to the job that creates or updates labels and issues, with unrelated permissions read-only.

**Focused verification gate**: Inspect workflow permissions and assert the label/issue job has issues: write only where required.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 15. Thread `PRRT_kwDOTNkC_s6PGKf_` — Gate sensitive evidence uploads.

- File: `docs/issues/32-workflow-templates.md`
- Line: 50
- Finding basis: Existing review finding “Gate sensitive evidence uploads.” at `docs/issues/32-workflow-templates.md:50`.

**Normative resolution**: At `docs/issues/32-workflow-templates.md:50`, When containsSensitiveInput is true, the workflow SHALL stop before uploading evidence and SHALL retain only a redacted manifest.

**Focused verification gate**: Run sensitive-evidence path and assert no artifact upload request occurs and the manifest is redacted.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 16. Thread `PRRT_kwDOTNkC_s6PGKgH` — Cover T13 and T14 too.

- File: `docs/issues/35-security-docs.md`
- Line: 34
- Finding basis: Existing review finding “Cover T13 and T14 too.” at `docs/issues/35-security-docs.md:34`.

**Normative resolution**: At `docs/issues/35-security-docs.md:34`, The security traceability matrix SHALL contain rows T1 through T14 with implementation, test, and residual-risk reference for each.

**Focused verification gate**: Check the matrix and require T1 through T14 each have module, test, and residual-risk cells.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 17. Thread `PRRT_kwDOTNkC_s6PGKgM` — Parameterize the release validation for the fallback package name.

- File: `docs/issues/36-release-pipeline.md`
- Line: 44
- Finding basis: Existing review finding “Parameterize the release validation for the fallback package name.” at `docs/issues/36-release-pipeline.md:44`.

**Normative resolution**: At `docs/issues/36-release-pipeline.md:44`, Release validation SHALL use one packageName variable for availability, install, smoke execution, and assertions, including the scoped fallback.

**Focused verification gate**: Run release validation with unscoped and scoped package names and assert every command uses selected packageName.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 18. Thread `PRRT_kwDOTNkC_s6PGLHY` — Strip secrets before spawning audit commands

- File: `docs/issues/21-target-provisioner.md`
- Line: 44
- Finding basis: Existing review finding “Strip secrets before spawning audit commands” at `docs/issues/21-target-provisioner.md:44`.

**Normative resolution**: At `docs/issues/21-target-provisioner.md:44`, Audit command targets SHALL receive the sanitized environment removing the four configured credential sources before spawn.

**Focused verification gate**: Inspect audit child environment and assert all configured credential names are absent.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 19. Thread `PRRT_kwDOTNkC_s6PGLHZ` — Disable checkout credential persistence in fix workflows

- File: `docs/issues/32-workflow-templates.md`
- Line: 39
- Finding basis: Existing review finding “Disable checkout credential persistence in fix workflows” at `docs/issues/32-workflow-templates.md:39`.

**Normative resolution**: At `docs/issues/32-workflow-templates.md:39`, Every fix workflow checkout SHALL set persist-credentials: false before the token is exposed to the action.

**Focused verification gate**: Run fix workflow and inspect git config after checkout; assert no persisted checkout credential.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 20. Thread `PRRT_kwDOTNkC_s6PGLHb` — Make duplicate fingerprints stable after fixes

- File: `docs/DESIGN.md`
- Line: 336
- Finding basis: Existing review finding “Make duplicate fingerprints stable after fixes” at `docs/DESIGN.md:336`.

**Normative resolution**: At `docs/DESIGN.md:336`, Duplicate findings SHALL use file path, stable DOM selector, source start/end range, and rule ID as identity; occurrence index SHALL not be used.

**Focused verification gate**: Remove the first duplicate DOM finding and rerun; assert the remaining finding keeps its fingerprint.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 21. Thread `PRRT_kwDOTNkC_s6PGLHe` — Expose per-rule patches before grouping commits

- File: `docs/issues/17-fix-command.md`
- Line: 42
- Finding basis: Existing review finding “Expose per-rule patches before grouping commits” at `docs/issues/17-fix-command.md:42`.

**Normative resolution**: At `docs/issues/17-fix-command.md:42`, The fix handoff SHALL pass Map<ruleId, FilePatch[]> to the Git engine, and each commit SHALL contain only patches for its rule ID.

**Focused verification gate**: Create two rule findings in one file and assert generated commits contain disjoint per-rule patches.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 22. Thread `PRRT_kwDOTNkC_s6PGLHf` — Do not require 404 after the first npm publish

- File: `docs/issues/36-release-pipeline.md`
- Line: 42
- Finding basis: Existing review finding “Do not require 404 after the first npm publish” at `docs/issues/36-release-pipeline.md:42`.

**Normative resolution**: At `docs/issues/36-release-pipeline.md:42`, The npm availability check SHALL run only before the first publication; later releases SHALL verify ownership of the existing package and SHALL not require a 404.

**Focused verification gate**: Simulate first publication and subsequent release; assert only the first performs 404 checking.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 23. Thread `PRRT_kwDOTNkC_s6PGLHi` — Mark WCAG-less UX findings as best-practice

- File: `docs/issues/28-llm-ux-analyst.md`
- Line: 55
- Finding basis: Existing review finding “Mark WCAG-less UX findings as best-practice” at `docs/issues/28-llm-ux-analyst.md:55`.

**Normative resolution**: At `docs/issues/28-llm-ux-analyst.md:55`, A finding with no valid WCAG success-criterion reference SHALL set bestPractice true before schema validation.

**Focused verification gate**: Create an advisory finding without WCAG refs and assert schema validation passes with bestPractice true.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 24. Thread `PRRT_kwDOTNkC_s6PGLHk` — Keep fix-pr from dirtying the checked-out branch

- File: `docs/issues/30-pr-engine.md`
- Line: 55
- Finding basis: Existing review finding “Keep fix-pr from dirtying the checked-out branch” at `docs/issues/30-pr-engine.md:55`.

**Normative resolution**: At `docs/issues/30-pr-engine.md:55`, The fix-pr flow SHALL run in a temporary isolated worktree and SHALL leave the checked-out source branch unchanged.

**Focused verification gate**: Run non-dry fix-pr and assert source checkout status is unchanged while the isolated worktree changes.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 25. Thread `PRRT_kwDOTNkC_s6PGLHl` — Do not move the stable v1 tag for prereleases

- File: `docs/issues/36-release-pipeline.md`
- Line: 35
- Finding basis: Existing review finding “Do not move the stable v1 tag for prereleases” at `docs/issues/36-release-pipeline.md:35`.

**Normative resolution**: At `docs/issues/36-release-pipeline.md:35`, The stable v1 tag SHALL move only for a non-prerelease version; prereleases SHALL use the separate v1-rc tag.

**Focused verification gate**: Run prerelease and stable releases; assert only stable moves v1 and prerelease moves v1-rc.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 26. Thread `PRRT_kwDOTNkC_s6PGLHm` — Resolve the GitHub token in the action entrypoint

- File: `docs/issues/31-github-action.md`
- Line: 32
- Finding basis: Existing review finding “Resolve the GitHub token in the action entrypoint” at `docs/issues/31-github-action.md:32`.

**Normative resolution**: At `docs/issues/31-github-action.md:32`, The action metadata token input SHALL default to empty, and the entrypoint SHALL use the runner GITHUB_TOKEN when input is empty and fail closed when neither exists.

**Focused verification gate**: Invoke action with empty input plus runner token, then with neither; assert fallback use then fail closed.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 27. Thread `PRRT_kwDOTNkC_s6PGLHo` — Add checkout and Node setup before npm commands

- File: `docs/issues/01-repo-scaffold.md`
- Line: 66
- Finding basis: Existing review finding “Add checkout and Node setup before npm commands” at `docs/issues/01-repo-scaffold.md:66`.

**Normative resolution**: At `docs/issues/01-repo-scaffold.md:66`, Each CI job SHALL run checkout and the requested setup-node matrix step before npm ci.

**Focused verification gate**: Inspect job order and run a matrix job; assert checkout and setup-node precede npm ci.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 28. Thread `PRRT_kwDOTNkC_s6PGLHs` — Define the flow-step schema in the config issue

- File: `docs/issues/25-flow-runner.md`
- Line: 31
- Finding basis: Existing review finding “Define the flow-step schema in the config issue” at `docs/issues/25-flow-runner.md:31`.

**Normative resolution**: At `docs/issues/25-flow-runner.md:31`, Issue 02 SHALL own and validate the closed flow-step union and payload schema; Issue 25 SHALL consume that validated schema.

**Focused verification gate**: Pass an invalid flow step through config load and assert Issue 02 rejects it before flow execution.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 29. Thread `PRRT_kwDOTNkC_s6PGLHu` — Redact token-shaped text from Markdown reports

- File: `docs/issues/10-markdown-report.md`
- Line: 88
- Finding basis: Existing review finding “Redact token-shaped text from Markdown reports” at `docs/issues/10-markdown-report.md:88`.

**Normative resolution**: At `docs/issues/10-markdown-report.md:88`, Markdown report rendering SHALL apply token/secret redaction before Markdown escaping, replacing matched material with ***.

**Focused verification gate**: Render a report containing a token-shaped string and assert output contains *** rather than the token.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 30. Thread `PRRT_kwDOTNkC_s6PGLHw` — Create .gitignore when initializing a repo without one

- File: `docs/issues/02-config-loader.md`
- Line: 74
- Finding basis: Existing review finding “Create .gitignore when initializing a repo without one” at `docs/issues/02-config-loader.md:74`.

**Normative resolution**: At `docs/issues/02-config-loader.md:74`, Repository initialization SHALL create .gitignore when absent and SHALL add .a11ybot/ before writing bot artifacts.

**Focused verification gate**: Initialize a repository without .gitignore and assert it is created with .a11ybot/.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 31. Thread `PRRT_kwDOTNkC_s6PGLHy` — Fix viewport-only unsafe directives instead of skipping

- File: `docs/issues/14-fixers-html.md`
- Line: 47
- Finding basis: Existing review finding “Fix viewport-only unsafe directives instead of skipping” at `docs/issues/14-fixers-html.md:47`.

**Normative resolution**: At `docs/issues/14-fixers-html.md:47`, A viewport containing only removable unsafe directives SHALL return an edit that removes the unsafe content or attribute and SHALL not return fix_skipped.

**Focused verification gate**: Run viewport containing only unsafe directives and assert an edit removes them.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

### 32. Thread `PRRT_kwDOTNkC_s6PGLH0` — Include all threat rows in the security traceability doc

- File: `docs/issues/35-security-docs.md`
- Line: 32
- Finding basis: Existing review finding “Include all threat rows in the security traceability doc” at `docs/issues/35-security-docs.md:32`.

**Normative resolution**: At `docs/issues/35-security-docs.md:32`, The security traceability document SHALL cover all T1 through T14 rows, including state-changing live flows T13 and secret propagation to child processes T14.

**Focused verification gate**: Check security document for T1-T14 and assert T13/T14 have module, test, and residual-risk treatment.

**Completion boundary**: This is a design-level response contract only. Resolve this thread only after its focused gate, applicable specialist handoff, and repository full validation are terminal success for the current head/base identity.

## Merge boundary

- `gate-task-evaluator` must re-fetch PR state, current head/base, required-check inventory, review decision, unresolved thread state, policy version, and merge candidate immediately before any merge mutation.
- `github_mergeable` and a successful CodeRabbit status are not merge authorization.
- This task permits at most one PR Bot review. This artifact authorizes no Bot trigger or rerun.
