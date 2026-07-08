# ADR-007: Stack — TypeScript strict / Node.js >= 22 / ESM / vitest / npm

- Status: accepted (2026-07-08; defaults proposed by design, no owner objection)

## Decision

| Concern | Choice | Rationale |
|---|---|---|
| Language | TypeScript, `strict: true` | Ecosystem alignment (ESLint/Playwright are TS-native); type safety for weaker implementation agents |
| Runtime | Node.js **>= 22** | Node 20 reached EOL 2026-04; 22 is the oldest maintained LTS at design time (2026-07) |
| Module format | ESM only (`"type": "module"`) | Current ecosystem default; avoids dual-package hazards |
| Package manager | npm + committed `package-lock.json` | Zero extra tooling for contributors and CI |
| Tests | vitest | Fast, TS-native, snapshot support for mapping tables |
| Build | tsup for library/CLI; bundled `dist/` for the GitHub Action | Simple, single-config |
| Repo shape | Single package (no monorepo) | One publishable unit + `action.yml` at repo root; monorepo overhead not justified in v1 |
| Own-code lint | ESLint 9 flat config + Prettier | Same major as the embedded engine (one ESLint version in the tree) |

## Consequences

- `engines.node: ">=22"` in package.json; CI matrix pins Node 22 and 24.
- The embedded ESLint 9.x pin (ADR-002) doubles as the dev-lint version — a single
  ESLint version in `node_modules` prevents resolution ambiguity.
