# Title

Repository scaffold: TypeScript/ESM package, lint, test, CI

## Summary

Create the buildable, testable, lintable skeleton of the `a11y-bot` npm package so
every later issue lands on a working toolchain.

## Context

Repository contains only README and `docs/` (design documents) — no toolchain or
source yet. Stack decisions are fixed by ADR-007: TypeScript strict, Node >= 22,
ESM-only, npm, vitest, tsup, ESLint 9 flat config + Prettier for our own code.
Single package, no monorepo.

## Scope

- `package.json`, `tsconfig.json`, build/test/lint wiring, directory skeleton,
  CI workflow for this repo, LICENSE, .gitignore, .editorconfig.
- Core primitives with no dependencies that issues 02/03/04 all need:
  `src/core/errors.ts` (`ConfigError`, `EnvError`, `InternalError` classes with
  `hint?: string`, `docsUrl?: string` fields) and `src/core/exit-codes.ts`
  (constants 0/1/2/3/4 per DESIGN.md §12.3). Classes only — the top-level
  handler that maps them to exit codes is issue 04.
- No product logic beyond that. Placeholder CLI entry that prints name/version.

## Detailed Requirements

1. `package.json`:
   - `"name": "a11y-bot"`, `"version": "0.0.0"`, `"type": "module"`,
     `"license": "MIT"`, `"engines": { "node": ">=22" }`,
     `"bin": { "a11y-bot": "./dist/cli/index.js" }`,
     `"files": ["dist", "schemas"]` (issue 31 appends `"action.yml"` when it
     creates that file).
   - `package-lock.json` is committed (DESIGN §14.2 T10) and CI uses `npm ci`.
   - Scripts: `build` (tsup), `test` (vitest run), `test:watch`, `lint`
     (eslint .), `format` (prettier --check .), `typecheck` (tsc --noEmit).
2. `tsconfig.json`: `strict: true`, `module: NodeNext`, `moduleResolution:
   NodeNext`, `target: ES2023`, `noUncheckedIndexedAccess: true`,
   `exactOptionalPropertyTypes: true`, `outDir: dist` (tsup handles emit; tsc is
   typecheck-only).
3. tsup config: entry `src/cli/index.ts`, format esm, `banner: #!/usr/bin/env node`
   on the CLI entry, sourcemaps, node22 platform target.
4. Directory skeleton with `.gitkeep` or index stubs — the full DESIGN.md §5
   tree: `src/cli/commands/`, `src/config/`, `src/core/messages/`,
   `src/static/adapters/`, `src/static/rule-maps/`, `src/fix/fixers/{html,jsx,vue}/`,
   `src/llm/providers/`, `src/llm/fixers/`, `src/audit/probes/`, `src/report/`,
   `src/github/`, `action/`, `schemas/`, `fixtures/`, `examples/workflows/`.
5. Version plumbing: tsup `define` injects `__A11YBOT_VERSION__` from
   package.json; `src/core/version.ts` exports
   `export const VERSION: string = __A11YBOT_VERSION__;` plus an ambient
   declaration (`src/types/build.d.ts`); vitest runs without tsup, so the
   declaration provides a dev fallback
   (`globalThis.__A11YBOT_VERSION__ ?? "0.0.0-dev"` pattern — document in code).
   `src/cli/index.ts` stub prints `a11y-bot ${VERSION}` and exits 0 (replaced in
   issue 04).
6. Own-code lint: `eslint.config.js` flat config (ESLint 9.x pinned — same major
   as the embedded engine), `@typescript-eslint` recommended, Prettier via
   config-prettier (no format-in-lint). `.prettierrc` with default options.
7. `.gitignore`: node_modules, dist, coverage, `.a11ybot/` (reports, evidence,
   baseline are user-repo artifacts, but this repo's own test runs write there
   too).
8. CI (`.github/workflows/ci.yml`): triggers `push` to main + `pull_request`;
   job matrix Node 22 and 24; steps: `npm ci`, `npm run lint`, `npm run
   typecheck`, `npm run build`, `npm test`. Workflow-level
   `permissions: { contents: read }`.
9. `LICENSE`: MIT, copyright holder = repository owner.
10. One sample vitest test (`src/core/__tests__/smoke.test.ts`) asserting the
    built CLI output (`node dist/cli/index.js`, spawned) equals
    `a11y-bot ${package.json version}` — this exercises the injected constant.

## Acceptance Criteria

- [ ] `npm ci && npm run lint && npm run typecheck && npm run build && npm test`
      all succeed locally on Node 22.
- [ ] `node dist/cli/index.js` prints `a11y-bot 0.0.0` and exits 0.
- [ ] `npx a11y-bot` works after `npm pack` + install of the tarball in a temp dir.
- [ ] CI workflow is green on the PR that introduces it and runs on both Node
      versions with `contents: read` permissions only.
- [ ] Repository layout matches DESIGN.md §5 (all paths in requirement 4 exist).
- [ ] `package-lock.json` committed; `git ls-files package-lock.json` non-empty.
- [ ] `src/core/errors.ts` exports the three error classes and
      `src/core/exit-codes.ts` exports the five constants (unit test imports
      and asserts values 0–4).

## Validation

```bash
npm ci
npm run lint && npm run typecheck && npm run build && npm test
node dist/cli/index.js
npm pack --dry-run   # confirm files list contains only dist, schemas, README, LICENSE, package.json
```

## Dependencies

None (first issue).

## Non-goals

Any command logic, config parsing, error-handler/formatter (issue 04),
publishing setup (issue 36), Action bundle (issue 31).

## Design References

DESIGN.md §5 (layout), §16 (test stack), §12.3 (exit-code constants), §15
(error class taxonomy), §14.2 T10 (lockfile); ADR-007 (stack), ADR-008 (license).
