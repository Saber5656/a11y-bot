# Title

CLI skeleton, logger, error taxonomy, exit codes

## Summary

Build the `a11y-bot` command-line surface (commander), the shared logger, the
error class hierarchy, and the global exit-code contract that every command uses.

## Context

Commands `scan|fix|audit` are implemented by later issues; this issue delivers the
frame they plug into, plus fully working `init` (function from issue 02),
`--help`, and `--version`. DESIGN.md §12.3 and §15 are normative.

## Scope

- `src/cli/index.ts` (program definition, global flags, top-level error handler).
- `src/cli/commands/{init,scan,fix,audit}.ts` (init: real; others: stub that
  exits 3 "not implemented" until their issues land — stubs replaced in 09/17/27).
- `src/core/logger.ts`, `src/core/errors.ts`, `src/core/exit-codes.ts`,
  `src/core/run-context.ts`.

## Detailed Requirements

1. Program: `a11y-bot <command>` with commands `init`, `scan`, `fix`, `audit`.
   Global flags: `--config <path>`, `--quiet`, `--verbose`, `--no-color`,
   `--version`, `--help`. Unknown command/flag → usage error, exit 2.
2. Logger:
   - Levels debug/info/warn/error; default info; `--quiet` = error-only;
     `--verbose` or `A11YBOT_LOG=debug` = debug.
   - Colors auto-disabled when `!process.stdout.isTTY` or `NO_COLOR` or
     `--no-color`.
   - **Redaction filter**: any logged string is scrubbed for values of
     `process.env[config.llm.apiKeyEnv]`, `OPENAI_API_KEY`, `GITHUB_TOKEN`,
     `GH_TOKEN` (exact-value replace with `***`), plus regex for GitHub token
     shapes (`gh[pousr]_[A-Za-z0-9]{20,}`). Unit-tested.
   - All diagnostics to stderr; machine output (JSON reports to stdout when
     `--format json` without `--output`) stays clean on stdout.
3. Errors (`src/core/errors.ts`): `ConfigError`(2), `EnvError`(4),
   `InternalError`(3), each with `hint?: string` and `docsUrl?: string`.
   Top-level handler maps error class → exit code per DESIGN §12.3, prints
   `error: <message>` + optional `hint:` line; stack trace only at debug level.
   Unexpected (non-taxonomy) exceptions → InternalError treatment.
4. `run-context.ts`: assembles `{ config, logger, cwd, runId }` once per
   invocation; `runId = <YYYYMMDDTHHmmssZ>-<8 hex random>`.
5. Exit code constants exported from one module; commands may not call
   `process.exit` directly except via the shared `exitWith(code)` helper
   (lint rule or code-review note in the issue for the implementer).
6. `--version` reads version injected at build time (tsup define), matching
   package.json.

## Acceptance Criteria

- [ ] `a11y-bot --help` lists 4 commands; `a11y-bot init` works end-to-end;
      `scan/fix/audit` stubs exit 3 with "not implemented (see issue NN)".
- [ ] Exit codes: usage error→2, ConfigError→2, EnvError→4, InternalError→3
      (integration tests spawning the built CLI).
- [ ] Redaction test: with `GITHUB_TOKEN=ghp_XXXX…` set and a debug log echoing
      env-derived strings, output contains `***` and never the token.
- [ ] stdout/stderr separation test: `--format json` (once 09 lands, prepare the
      helper now) keeps stdout parseable.
- [ ] No direct `process.exit` outside the helper (grep check documented).

## Validation

```bash
npm run build
node dist/cli/index.js --help
node dist/cli/index.js scan; echo "exit=$?"        # expect 3 (stub)
GITHUB_TOKEN=ghp_0123456789abcdefghij node dist/cli/index.js --verbose init
npm test -- src/cli src/core
```

## Dependencies

01, 02.

## Non-goals

Command business logic (09, 17, 27); PR engine token use (30).

## Design References

DESIGN.md §12.3 (exit codes), §15 (errors), §6.4 (env), §14.2 T3 (redaction).
