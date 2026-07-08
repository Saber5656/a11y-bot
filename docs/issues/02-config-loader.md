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
- `src/config/init.ts` + CLI subcommand `init` (wired fully in issue 04; here a
  callable function + minimal registration).
- `schemas/a11ybot.schema.json` generation script (`npm run gen:schemas`).

## Detailed Requirements

1. Implement the schema exactly as DESIGN.md §6.2/§6.3 with these validation
   rules:
   - `version` literal `1`; anything else → ConfigError "unsupported config version".
   - `.strict()` on every object (unknown key → error naming the key and its path).
   - `audit.targets[]`: discriminated union on exactly one of `url` / `staticDir`
     / `command`; `name` matches `^[a-z0-9-]{1,40}$` and is unique across targets.
   - `command` variant requires `port`; `readyPath` defaults `/`;
     `readyTimeoutMs` defaults 60000.
   - `flows[].target` must reference an existing target name; `flows[].name`
     same pattern/uniqueness as targets.
   - `report.failOn` ∈ severity enum; `fix.classes` ⊆ {auto_safe, auto_review}.
   - `llm.apiKeyEnv` must match `^[A-Z][A-Z0-9_]*$`.
2. Loader behavior:
   - Search order: `--config <path>` (error if missing) → `.a11ybot.yml` →
     `.a11ybot.yaml` in cwd. No config file → all defaults (valid).
   - YAML via `yaml` package, `version: 1.2` core schema, no custom tags,
     max file size 256 KiB (ConfigError beyond).
   - Overlay precedence: CLI flags > env > file > defaults (DESIGN §6.1). Env
     mappings: `A11YBOT_LOG`; secrets are NOT config values — the loader stores
     only the env *name* (`apiKeyEnv`), never reads the key itself (consumers do).
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
- [ ] Unknown key, bad target union, duplicate names, bad `failOn` each produce a
      ConfigError that names the exact YAML path; multiple errors reported together.
- [ ] No config file → defaults object identical to DESIGN §6.2 defaults (snapshot test).
- [ ] `init` creates a config that the loader parses without errors; second run
      without `--force` exits 2 with a clear message.
- [ ] `schemas/a11ybot.schema.json` committed and validated fresh in CI.

## Validation

```bash
npm test -- src/config
npm run gen:schemas && git diff --exit-code schemas/
node dist/cli/index.js init && node -e "..."   # loader smoke via test script
```

## Dependencies

01.

## Non-goals

Reading LLM keys (issue 18), interpreting scan/audit options (their subsystems),
full CLI flag surface (issue 04).

## Design References

DESIGN.md §6 (all), §12.3 (exit 2), §14.3 (secure defaults); ADR-004 (key via env
name only).
