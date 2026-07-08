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
- `src/static/eslint-runner.ts` — engine lifecycle + execution.
- Dependency pins in package.json: `eslint@^9` (latest 9.x),
  `eslint-plugin-jsx-a11y@^6.10`, `eslint-plugin-vuejs-accessibility@^2.5`,
  `vue-eslint-parser`, `@html-eslint/eslint-plugin@^0.63`, `@html-eslint/parser`,
  `typescript-eslint` (parser only), `globby`.

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
   - Execution: `lintFiles(batch)` per profile; per-file fatal parse errors are
     converted to `static/bot/parse-error` findings (message includes line/col),
     never abort the run.
   - Timeout guard: a profile run exceeding 120 s → InternalError with hint
     (protects CI hangs; DESIGN §14.2 T12).
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

DESIGN.md §8.1, §14.2 T12, §2.3 U2; ADR-002; research/2026-07-static-lint-engines.md.
