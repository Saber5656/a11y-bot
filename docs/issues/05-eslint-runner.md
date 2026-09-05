# Title

Embedded ESLint 9 runner and file discovery

## Summary

Implement the isolated ESLint engine wrapper (pinned 9.x) and the repository file
discovery that feeds it, producing raw `LintResult`s for the adapters.

## Context

ADR-002: we embed ESLint as an internal engine; the user's own ESLint setup must
never leak in. Research doc `2026-07-static-lint-engines.md` fixes versions and
the ESLint-10 incompatibility rationale. DESIGN.md §8.1 is normative.

## Scope

- `src/static/discover.ts` — file discovery.
- `src/static/eslint-runner.ts` — engine lifecycle + execution (worker-based).
- `src/static/rule-maps/types.ts` — the shared `RuleMapRow` type used by all
  adapters (06–08): `{ upstream: string; unified: string; wcag: WcagRef[];
  bestPractice?: true; defaultSeverity: Severity; fixability: Fixability;
  enabled: boolean; configGated?: string; messageId: string; docsUrl: string }`.
- `applyRuleControls(findings, config)` — DESIGN §8.3 rule-control semantics
  (`off` drops; `warn` remaps severity to `minor` and marks non-gating; `error`
  keeps registry severity and gates), shared by scan/fix/audit commands.
- Dependency pins in package.json (2026-07-08 versions; implementer bumps
  within the same major): `eslint@^9.39` (NOT 10 — version-lock test),
  `eslint-plugin-jsx-a11y@^6.10.2`, `eslint-plugin-vuejs-accessibility@^2.5.0`,
  `vue-eslint-parser@^10.4.1`, `@html-eslint/eslint-plugin@~0.63.0`,
  `@html-eslint/parser@~0.63.0`, `typescript-eslint@^8` (parser only), `globby@^14`.

## Detailed Requirements

1. Discovery:
   - Input: `scan.include` / `scan.exclude` globs + optional CLI `[paths]`
     narrowing; POSIX-normalized repo-relative output paths.
   - Always-excluded: `node_modules/`, `.git/`, `dist/`, `build/`, `out/`,
     `coverage/`, `.next/`, `.nuxt/`, `vendor/`; plus `.gitignore` patterns
     (use `globby` gitignore support).
   - Symlinks not followed. Files > 1 MiB skipped and reported via finding
     `static/bot/file-skipped` (registry entry from issue 03 message
     `bot.file-skipped`).
   - Deterministic ordering (lexicographic) for stable output.
   - Enforce `scan.maxFiles`: exceeding → ConfigError naming the count and cap.
2. Runner:
   - Three engine profiles (html/jsx/vue) per DESIGN §8.1 table, each a
     `new ESLint({...})` with: `overrideConfigFile: true`, in-memory flat config,
     `allowInlineConfig: false` (inline `eslint-disable` comments in user files
     must NOT suppress findings — a11y-bot has its own rule-control mechanism),
     `cache: false`, `cwd` = repo root.
   - JSX/TSX profile: `typescript-eslint` parser without type-aware project
     service (no `parserOptions.project`) — fast, no tsconfig requirement.
   - Rule sets initially empty here; adapters (06–08) contribute
     `{ plugin, rules, parser }` fragments via a registration API
     `defineProfile(name, fragment)` so this issue is testable with a dummy rule.
   - **Execution model (T12 guard)**: each profile runs inside a
     `node:worker_threads` Worker (ESLint instantiated in the worker); the host
     enforces a 120 s hard timeout per profile via `worker.terminate()` →
     InternalError with hint "possible pathological input; try excluding the
     last-logged file". This is what makes a same-thread parser hang
     preemptible. Files are passed as paths; results as structured clones.
   - Output contract: `runProfiles(files: DiscoveredFiles): Promise<{
     lintResults: ESLintResult[]; botFindings: Finding[] }>` where
     `DiscoveredFiles = { byProfile: Record<"html"|"jsx"|"vue", string[]>;
     botFindings: Finding[] }` (discovery contributes `file-skipped`; the
     runner appends `parse-error` findings).
   - Per-file fatal parse errors are converted to `static/bot/parse-error`
     findings and never abort the run.
   - This issue registers the two bot rules in the registry:
     `static/bot/file-skipped` and `static/bot/parse-error` — both
     `{ engine: "static", wcag: [], bestPractice: true, defaultSeverity:
     "minor", fixability: "none", enabled: true }` (messages from issue 03's
     catalog).
3. Version-lock test: assert `require('eslint/package.json').version` starts with
   `9.` so an accidental major bump fails CI (ADR-002 watchpoint).

## Acceptance Criteria

- [ ] Discovery honors include/exclude/built-ins/.gitignore, skips >1 MiB with a
      finding, stable ordering (fixture-based tests).
- [ ] `allowInlineConfig: false` verified: fixture with `<!-- eslint-disable -->`
      / `{/* eslint-disable */}` still yields findings.
- [ ] User-repo ESLint config (`eslint.config.js` fixture with conflicting rules)
      demonstrably ignored.
- [ ] Fatal parse error fixture yields `static/bot/parse-error` finding and other
      files still lint.
- [ ] T12 guards tested: (a) a test-hook worker that never returns is terminated
      at the timeout and surfaces InternalError; (b) `scan.maxFiles` overflow →
      ConfigError naming count and cap; (c) >1 MiB file skipped with finding.
- [ ] `applyRuleControls` unit-tested: `off` drops, `warn` → severity `minor` +
      non-gating, `error` keeps registry severity.
- [ ] ESLint 9 version-lock test present and green.

## Validation

```bash
npm test -- src/static
```

## Dependencies

02, 03.

## Non-goals

Rule mapping to unified findings (06–08); scan CLI wiring (09).

## Design References

DESIGN.md §8.1, §8.3 (rule controls), §15 (error taxonomy, parse-error
continuation), §14.2 T12, §2.3 U2; ADR-002;
research/2026-07-static-lint-engines.md.
