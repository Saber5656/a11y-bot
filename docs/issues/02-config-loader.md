# Title

Config schema, loader, and `init` command

## Summary

Implement `.a11ybot.yml` parsing/validation with zod, config precedence, JSON
Schema generation, and the `a11y-bot init` starter-config generator.

## Context

Every subsystem reads configuration through this module. DESIGN.md §6 is the
normative schema, including defaults, the `audit.targets[]` discriminated union,
and env-var rules. Unknown keys must be hard errors.

## Scope

- `src/config/schema.ts` (zod schemas + inferred TS types, exported as `A11ybotConfig`).
- `src/config/load.ts` (file discovery, YAML parse, env/flag overlay, defaults).
- `src/config/init.ts`: exported function `initConfig(opts: { cwd: string;
  force: boolean }): Promise<{ path: string }>` — **no CLI registration here**;
  issue 04 wires the `init` command to this function.
- `schemas/a11ybot.schema.json` generation script (`npm run gen:schemas`).

## Detailed Requirements

1. Implement the schema exactly as DESIGN.md §6.2/§6.3 with these validation
   rules:
   - `version` literal `1`; anything else → ConfigError "unsupported config version".
   - `.strict()` on every object (unknown key → error naming the key and its path).
   - `audit.targets[]`: `z.union` of three strict object schemas plus a
     `superRefine` asserting exactly one of `url` / `staticDir` / `command` is
     present (zod's `discriminatedUnion` is NOT applicable — there is no shared
     discriminator key); `name` matches `^[a-z0-9-]{1,40}$` and is unique
     across targets.
   - `github.branchPrefix` must match `^a11y-bot\/[a-z0-9._\/-]*$`
     (DESIGN §6.2/§13.1 allowlist consistency); `github.closeEmptyPr`
     (boolean, default true) and `github.commitIdentity`
     (`{ name: string(1..64), email: string(valid email shape) }`, defaults per
     §6.2) are part of the schema.
   - `command` variant requires `port`; `readyPath` defaults `/`;
     `readyTimeoutMs` defaults 60000.
   - `flows[].target` must reference an existing target name; `flows[].name`
     same pattern/uniqueness as targets; `flows[].viewport` (optional) must
     reference an `audit.viewports[].name`.
   - `audit.viewports[].name` matches `^[a-z0-9-]{1,20}$` (evidence path
     component — DESIGN §14.2 T11); `audit.includeIncomplete`,
     `audit.llmAnalysis` (booleans, default false), `audit.keepRuns`
     (int ≥ 0, default 3), `llm.capabilities` (null or `{ vision: boolean }`),
     and `audit.targets[].readyTimeoutMs` for the url variant are all part of
     the §6.2 schema this issue implements.
   - `report.failOn` ∈ severity enum `critical|serious|moderate|minor`
     (DESIGN §7.1); `fix.classes` ⊆ {auto_safe, auto_review}.
   - `llm.apiKeyEnv` must match `^[A-Z][A-Z0-9_]*$`.
2. Loader behavior:
   - Search order: `--config <path>` (error if missing) → `.a11ybot.yml` at the
     repo root (git toplevel if inside a git repo, else cwd) — exactly one
     filename, matching DESIGN §6.1. No config file → all defaults (valid).
   - YAML via `yaml` package, `version: 1.2` core schema, no custom tags,
     max file size 256 KiB (ConfigError beyond — DoS guard, DESIGN §14.2 T12).
   - Overlay precedence: CLI flags > env > file > defaults (DESIGN §6.1).
     Env division of labor (document in code): the loader maps `A11YBOT_LOG`
     only; `NO_COLOR` is consumed by the logger (issue 04); the LLM key env is
     read by the LLM client (issue 18); `GITHUB_TOKEN`/`GH_TOKEN` by the GitHub
     modules (issues 29/30). Secrets are NOT config values — the loader stores
     only the env *name* (`apiKeyEnv`) and never reads secret values.
   - Returns frozen (`Object.freeze`, deep) config object + `configHash`
     (sha256 of canonical JSON, used by evidence manifest).
   - All validation failures aggregate into one ConfigError listing every issue
     (path, message), exit code 2 (via issue 04's handler).
3. `init`:
   - Writes `.a11ybot.yml` with commented defaults (template string in code,
     kept in sync with schema by a test that parses the template).
   - Appends `.a11ybot/` to `.gitignore` if a `.gitignore` exists and lacks it.
   - Refuses to overwrite existing config without `--force`; prints created path.
4. Schema JSON generation: script converts zod → JSON Schema (draft 2020-12) via
   `zod-to-json-schema` (or zod v4 native, whichever the pinned zod supports) into
   `schemas/a11ybot.schema.json`; CI check fails if the committed file is stale
   (`npm run gen:schemas && git diff --exit-code schemas/`).

## Acceptance Criteria

- [ ] Valid configs (each target variant, flows, rule overrides) parse to typed
      objects with documented defaults filled.
- [ ] Unknown key, bad target union, duplicate names, bad `failOn`, bad
      `branchPrefix` each produce a ConfigError that names the exact YAML path;
      multiple errors reported together.
- [ ] No config file → defaults object identical to DESIGN §6.2 defaults (snapshot test).
- [ ] `initConfig()` creates a config the loader parses without errors; second
      call without `force` throws ConfigError; `.a11ybot/` appended to an
      existing `.gitignore` (DESIGN §14.2 T7).
- [ ] Security: with `A11YBOT_LLM_API_KEY`/`GITHUB_TOKEN` set in test env, no
      loader error message, thrown object, or debug output contains the values
      (planted-canary test); 256 KiB oversize config → ConfigError.
- [ ] `schemas/a11ybot.schema.json` committed and validated fresh in CI.

## Validation

```bash
npm test -- src/config
npm run gen:schemas && git diff --exit-code schemas/
```

## Dependencies

01.

## Non-goals

Reading LLM keys (issue 18), interpreting scan/audit options (their subsystems),
full CLI flag surface (issue 04).

## Design References

DESIGN.md §6 (all), §7.1 (severity enum), §11.5 (flow schema), §12.3 (exit 2),
§14.2 T3/T7/T12, §14.3 (secure defaults); ADR-004 (key via env name only).
