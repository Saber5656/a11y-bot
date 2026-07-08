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
- `src/cli/commands/{init,scan,fix,audit}.ts` (init: wires issue 02's
  `initConfig()`; others: stub that exits 3 "not implemented" until their
  issues land — stubs replaced in 09/17/27).
- `src/core/logger.ts`, `src/core/run-context.ts`, and the top-level
  error-handling formatter (the error CLASSES and exit-code constants already
  exist from issue 01 — this issue consumes them).

## Detailed Requirements

1. Program: `a11y-bot <command>` with commands `init`, `scan`, `fix`, `audit`.
   Global flags: `--config <path>`, `--quiet`, `--verbose`, `--no-color`,
   `--version`, `--help`. Unknown command/flag → usage error, exit 2.
2. Logger:
   - Levels debug/info/warn/error; default info. Level resolution precedence
     (flags > env, DESIGN §6.1): `--quiet` → error-only; `--verbose` → debug;
     both flags together → usage error exit 2; no flags → `A11YBOT_LOG`
     (`debug` | `info` | `quiet`; invalid value → warn once, use info).
     `--verbose` and `A11YBOT_LOG=debug` are exact synonyms everywhere
     (including stack-trace visibility below).
   - Colors auto-disabled when `!process.stdout.isTTY` or `NO_COLOR` or
     `--no-color`.
   - **Redaction filter**: any logged string is scrubbed for values of
     `process.env[config.llm.apiKeyEnv]`, `OPENAI_API_KEY`, `GITHUB_TOKEN`,
     `GH_TOKEN` (exact-value replace with `***`), plus regex for GitHub token
     shapes (`gh[pousr]_[A-Za-z0-9]{20,}`). Unit-tested.
   - All diagnostics to stderr; machine output (JSON reports to stdout when
     `--format json` without `--output`) stays clean on stdout.
3. Error handling: top-level handler maps the issue-01 error classes → exit
   codes per DESIGN §12.3 (`ConfigError`→2, `EnvError`→4, `InternalError`→3),
   prints `error: <message>`, optional `hint: <hint>` line, and optional
   `docs: <docsUrl>` line; stack trace only at debug level (--verbose /
   A11YBOT_LOG=debug). Unexpected (non-taxonomy) exceptions → InternalError
   treatment. Exit-code constants (incl. `1` = gating findings, used by
   09/12/27) are re-exported through the CLI layer so command issues import
   from one place.
4. `run-context.ts`: assembles `{ config, logger, cwd, runId, commandLine }`
   once per invocation; `runId = <YYYYMMDDTHHmmssZ>-<8 hex random>`;
   `commandLine` = `a11y-bot <argv.slice(2) joined>` for report reproduction
   footers (issue 10), with values of flags named `*token*`/`*key*` replaced by
   `***`.
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
- [ ] Redaction tests cover ALL of: `GITHUB_TOKEN`, `GH_TOKEN`,
      `OPENAI_API_KEY`, the configured `llm.apiKeyEnv` value, and pattern-based
      GitHub token shapes — each planted value appears as `***`, never raw
      (DESIGN §14.2 T3).
- [ ] `A11YBOT_LOG=quiet` suppresses info/warn; `--quiet --verbose` → exit 2;
      invalid `A11YBOT_LOG` value falls back to info with one warning.
- [ ] Error formatter prints hint and docs lines when present (fixture error).
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

01 (error classes, exit-code constants), 02 (`initConfig()`, config loading).

## Non-goals

Command business logic (09, 17, 27); PR engine token use (30); defining error
classes (01).

## Design References

DESIGN.md §12.3 (exit codes), §15 (errors), §6.4 (env), §14.2 T3 (redaction).
