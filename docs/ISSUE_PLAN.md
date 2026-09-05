# a11y-bot — v1 Issue Plan

Derived from [DESIGN.md](./DESIGN.md). Each issue below has a full draft in
`docs/issues/NN-short-title.md`; GitHub Issues are generated from those drafts and
are non-canonical derived artifacts.

## v1 completion statement

**v1 is complete when all 36 issues below are completed and validated.** At that
point the product delivers: static a11y scanning of HTML/JSX/TSX/Vue with unified
findings; deterministic + optional-LLM fixes opening idempotent GitHub PRs; a
Playwright-based runtime audit (axe, deterministic probes, scripted flows,
evidence bundle) with optional LLM UX analysis; console/JSON/Markdown/SARIF
reports with baseline gating; a GitHub Action wrapper with documented least-
privilege workflow templates; multi-provider LLM SDK support; an E2E-tested,
security-documented, npm-published `1.0.0` release. Anything not covered by these
issues is out of v1 scope (see Deferred v2) except newly discovered implementation
unknowns, which must be filed as new issues referencing the unknown they resolve.

## Issue list (recommended execution order)

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-repo-scaffold.md` | Repository scaffold: TypeScript/ESM package, lint, test, CI | 0 |
| 02 | `issues/02-config-loader.md` | Config schema, loader, and `init` command | 0 |
| 03 | `issues/03-finding-model.md` | Unified finding model, rule registry, fingerprints, message catalog | 0 |
| 04 | `issues/04-cli-skeleton.md` | CLI skeleton, logger, error taxonomy, exit codes | 0 |
| 05 | `issues/05-eslint-runner.md` | Embedded ESLint 9 runner and file discovery | 1 |
| 06 | `issues/06-adapter-jsx.md` | JSX/TSX adapter: jsx-a11y rule map to unified findings | 1 |
| 07 | `issues/07-adapter-vue.md` | Vue SFC adapter: vuejs-accessibility rule map | 1 |
| 08 | `issues/08-adapter-html.md` | HTML adapter: @html-eslint a11y rule map | 1 |
| 09 | `issues/09-scan-command.md` | `scan` command: orchestration, console/JSON output, gating | 1 |
| 10 | `issues/10-markdown-report.md` | Markdown report renderer (shared by PR body / step summary) | 1 |
| 11 | `issues/11-sarif-report.md` | SARIF 2.1.0 exporter for GitHub code scanning | 1 |
| 12 | `issues/12-baseline.md` | Baseline file and fail-on-new gating | 1 |
| 13 | `issues/13-fix-engine-core.md` | Fix engine core: plan/apply/verify state machine | 2 |
| 14 | `issues/14-fixers-html.md` | Deterministic fixer batch: HTML rules | 2 |
| 15 | `issues/15-fixers-jsx.md` | Deterministic fixer batch: JSX rules | 2 |
| 16 | `issues/16-fixers-vue.md` | Deterministic fixer batch: Vue rules | 2 |
| 17 | `issues/17-fix-command.md` | `fix` command: local apply, dry-run diff, summary | 2 |
| 18 | `issues/18-llm-client.md` | LLM client core: OpenAI-compatible provider, budget, config | 3 |
| 19 | `issues/19-llm-guardrails.md` | LLM guardrails: data framing, output validators, injection test suite | 3 |
| 20 | `issues/20-llm-alt-text-fixer.md` | LLM content fixer: alt text / frame title generation | 3 |
| 21 | `issues/21-target-provisioner.md` | Audit target provisioning: url / staticDir / command variants | 4 |
| 22 | `issues/22-axe-scan.md` | Playwright session + axe-core scan to unified findings | 4 |
| 23 | `issues/23-probe-focus-keyboard.md` | Probes: focus-order walk, focus visibility, keyboard reachability | 4 |
| 24 | `issues/24-probe-visual-structure.md` | Probes: reflow, target size, structure outline | 4 |
| 25 | `issues/25-flow-runner.md` | Scripted flow runner (config DSL) with per-step evidence | 4 |
| 26 | `issues/26-evidence-bundle.md` | Evidence bundle: layout, manifest, scrubbing, caps | 4 |
| 27 | `issues/27-audit-command.md` | `audit` command: orchestration, reports, gating | 4 |
| 28 | `issues/28-llm-ux-analyst.md` | LLM UX analyst over evidence bundle (advisory findings) | 4 |
| 29 | `issues/29-git-engine.md` | Git engine: bot branch, rule-grouped commits, allowlist guard | 5 |
| 30 | `issues/30-pr-engine.md` | PR engine: idempotent create/update, body, labels, token handling | 5 |
| 31 | `issues/31-github-action.md` | GitHub Action wrapper: action.yml, bundled dist, step summary | 5 |
| 32 | `issues/32-workflow-templates.md` | Workflow templates + permissions/fork-safety documentation | 5 |
| 33 | `issues/33-llm-multi-provider.md` | Multi-provider LLM SDK layer (provider registry, capability flags) | 6 |
| 34 | `issues/34-e2e-fixtures.md` | E2E fixture projects and full-pipeline CI tests | 6 |
| 35 | `issues/35-security-docs.md` | SECURITY.md, threat-model doc, permissions guide | 6 |
| 36 | `issues/36-release-pipeline.md` | Release pipeline: npm provenance publish, versioning, v1 tag | 6 |

## Implementation waves

| Wave | Theme | Issues | Exit criterion |
|---|---|---|---|
| 0 | Foundations | 01–04 | `a11y-bot --help` runs from a built package; config + finding model unit-tested |
| 1 | Static scan MVP | 05–12 | `scan` gates a fixture repo correctly in CI with all four report formats |
| 2 | Deterministic fixes | 13–17 | `fix --dry-run` produces verified diffs on fixtures; golden tests pass |
| 3 | LLM optional layer | 18–20 | With stub LLM: alt-text fix passes validators; without key: clean skip |
| 4 | Runtime audit | 21–28 | `audit` on fixture site yields evidence bundle + findings + (stubbed) UX analysis |
| 5 | GitHub integration | 29–32 | Scheduled workflow on a sandbox repo opens/updates the fix PR end-to-end |
| 6 | Providers, E2E, security, release | 33–36 | `1.0.0` published with provenance; E2E suite green; security docs complete |

Parallelism notes: waves 2 and 4 can proceed in parallel after wave 1 (disjoint
modules); wave 3 can run alongside wave 2 after issue 13. Two cross-wave edges
break full parallelism and are intentional: issue 28 (wave 4) needs 19 (wave 3),
and issues 31/32 (wave 5) need 27 (wave 4). Issues 29/30 need only 17. Wave 6
requires everything (36 additionally depends on 33 so the release cannot ship
without multi-provider support).

## Dependency table

| Issue | Depends on (blocking) |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | 01 |
| 04 | 01, 02 |
| 05 | 02, 03 |
| 06 | 05 |
| 07 | 05 |
| 08 | 05 |
| 09 | 04, 06, 07, 08 |
| 10 | 03, 09 |
| 11 | 03, 09 |
| 12 | 09 |
| 13 | 09 |
| 14 | 13, 08 |
| 15 | 13, 06 |
| 16 | 13, 07 |
| 17 | 10, 13, 14, 15, 16 |
| 18 | 02, 04 |
| 19 | 18 |
| 20 | 19, 13 |
| 21 | 02, 04 |
| 22 | 21, 03 |
| 23 | 22 |
| 24 | 22, 23 |
| 25 | 02, 21, 22 |
| 26 | 22, 23, 24, 25 |
| 27 | 26, 10, 12 |
| 28 | 10, 19, 26, 27 |
| 29 | 17 |
| 30 | 29, 10 |
| 31 | 09, 17, 27, 30 |
| 32 | 31 |
| 33 | 18, 19, 20, 28 |
| 34 | 17, 27, 31 |
| 35 | 30, 32 |
| 36 | 33, 34, 35 |

## Coverage: DESIGN.md sections → issues

| DESIGN.md section | Covered by |
|---|---|
| §5 Repository layout, §16 test infra bootstrap, §15/§12.3 error classes + exit-code constants | 01 |
| §6 Configuration (schema, env, init) | 02 |
| §3 Conformance target (WCAG metadata model), §7 Finding model, registry, fingerprint, messages | 03 (level/version surfaced in reports via 09/10) |
| §12.3 exit-code enforcement, §15 error handler/formatter, CLI surface, logger | 04 |
| §8.1 Engine + discovery, §8.3 rule-control mechanism (`applyRuleControls`) | 05 |
| §8.2 Adapters (rule maps; §8.3 overrides consumed per adapter) | 06 (JSX), 07 (Vue), 08 (HTML) |
| §12.1 console/json reporters + gate wiring | 09 |
| §12.1 markdown | 10 |
| §12.1 sarif (+ research SARIF notes) | 11 |
| §12.2 Baseline & gating | 12 |
| §9.1–9.2 Fix classes & state machine | 13 |
| §9.3 Fixer catalog | 14, 15, 16 |
| §9.4 fix command | 17 |
| §10.1 LLM client & budget | 18 |
| §10.2 Guardrails | 19 |
| §10.3 Alt-text fixer | 20 |
| §11.1 Target provisioning | 21 |
| §11.2–11.3 Browser session + axe | 22 |
| §11.4 Probes (focus/keyboard) | 23 |
| §11.4 Probes (reflow/target-size/structure) | 24 |
| §11.5 Flows | 25 |
| §11.6 Evidence bundle | 26 |
| audit orchestration + §12 gating for audit | 27 |
| §10.3 UX analyst | 28 |
| §13.1 Git engine | 29 |
| §13.2 PR engine | 30 |
| §13.3 Action | 31 |
| §13.3 Templates + §14 T4/T5 | 32 |
| §10.1 provider registry (wave 6) | 33 |
| §16 E2E layer, §17 budgets verification | 34 |
| §14 Security model docs, §18 release security | 35 |
| §18 Distribution & release | 36 |

Every normative DESIGN.md requirement is owned by exactly one issue; issues cite
their sections under "Design References".

## Validation strategy (product-wide)

1. **Per-issue**: acceptance criteria + listed validation commands must pass in CI
   before an issue is closed (weak-agent-executable: commands are given verbatim).
2. **Wave gates**: each wave's exit criterion (table above) is checked on the
   fixture projects before the next wave starts.
3. **Adapter completeness tests** guard plugin upgrades (fail on unmapped rules).
4. **Golden fixer tests** guarantee fix idempotence: applying a fixer twice yields
   no second diff; verify loop asserts no new findings.
5. **Adversarial LLM suite** (issue 19) runs on every CI build with a stub
   provider; real-provider smoke tests are manual, documented in 33.
6. **E2E (issue 34)**: `scan` → `fix --dry-run` → `audit` full pipeline on three
   fixture apps in CI; `31` adds an in-repo Action smoke workflow; `30` validated
   against a scratch GitHub repo (documented manual gate before release).
7. **Release gate (issue 36)**: all above green + SECURITY docs merged + npm
   provenance verified on a release-candidate publish to a scoped test name.

## Deferred v2 items

GitHub App / hosted mode; autonomous browser-agent audits; runtime→source fix
mapping; suggested-changes PR comments; site crawling; Svelte/Astro/Angular/server
templates; custom rule plugin API; localized output (`ja` catalog); multi-browser
matrix; static color-contrast approximation; markuplint engine evaluation;
organization dashboards / trend storage.

## Known unknowns (may create additional issues)

Tracked in DESIGN.md §2.3 (U1–U7). Resolution rule: when an unknown invalidates an
acceptance criterion, file a new `docs/issues/NN-*.md` (next free number)
referencing the unknown, update this plan's dependency table, then implement.
