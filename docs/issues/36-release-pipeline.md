# Title

Release pipeline: npm provenance publish, versioning, v1 tag

## Summary

Automate the release path — version tagging, npm publish with provenance,
Action major-tag maintenance, changelog — and execute the `1.0.0` release gate.

## Context

DESIGN.md §18. npm name `a11y-bot` verified available 2026-07-08 (known unknown
U6: re-verify at publish). Release security is part of the threat model (T10).

## Scope

- `.github/workflows/release.yml`, `CHANGELOG.md` seed, publish scripts,
  README distribution/usage sections, npm metadata polish.

## Detailed Requirements

1. Versioning: semver; source of truth = git tag `vX.Y.Z` pushed by the
   maintainer (manual gate — no auto-release); workflow triggers on tag push.
2. `release.yml`:
   - `permissions: { contents: write, id-token: write }` (provenance needs
     id-token; contents for the GitHub release + major-tag move);
   - jobs: verify (full CI suite reuse via `workflow_call` or duplicated steps
     — choose and document) → publish (`npm publish --provenance --access
     public`, `NPM_TOKEN` from repo secret; registry 2FA/automation-token setup
     documented for the maintainer, agent does not create tokens) → GitHub
     Release with changelog excerpt → move/force `v1` major tag to the release
     commit (documented action or script; this is the ONE permitted force
     operation outside `a11y-bot/*` because it targets tags owned by releases —
     add explicit note to threat model in this PR).
3. Pre-publish assertions in the workflow: `npm pack` file list matches
   allowlist (dist, schemas, action.yml, README, LICENSE, CHANGELOG); bin
   executes (`node dist/cli/index.js --version` equals tag); U6 name check
   (`npm view a11y-bot` 404 or owned by this account — else abort with
   fallback plan `@<owner>/a11y-bot` documented in README note).
4. Release gate checklist (executed for `1.0.0`, recorded in the release PR):
   - all 36 issues closed; E2E green on the release commit;
   - security checklist from issue 35 all green;
   - scratch-repo manual validation of `fix-pr` + scheduled workflow (30/32)
     re-run on the release candidate;
   - README quickstart validated by following it verbatim on a fresh machine
     container (documented transcript).
5. `CHANGELOG.md`: keep-a-changelog format seeded with `1.0.0` sections
   generated from wave summaries (manual curation allowed; no auto-generation
   dependency in v1).
6. README completion: badges (CI, npm), quickstart (init → scan → fix-pr),
   Action usage block, links to guides + security docs. README stays English;
   existing Japanese one-liner replaced (with the owner-approved English
   product statement + a short Japanese paragraph retained at the bottom for
   continuity — explicit owner sign-off on final README text before merge).

## Acceptance Criteria

- [ ] Tag-push dry run on a prerelease tag (`v1.0.0-rc.1`, `--tag next`)
      publishes to npm with provenance verified
      (`npm view a11y-bot@1.0.0-rc.1 --json` provenance field) — or to the
      scoped fallback if U6 fails, with docs updated accordingly.
- [ ] Pack-allowlist and version-match assertions demonstrated failing on a
      deliberate mismatch (negative test in a draft, reverted).
- [ ] `v1` major tag mechanism works (rc exercise) and threat-model note added.
- [ ] Release-gate checklist document committed and fully checked for 1.0.0.
- [ ] README final text owner-approved; quickstart transcript linked.

## Validation

```bash
npm pack --dry-run
# rc tag exercise per requirements 2–3
```

## Dependencies

34, 35.

## Non-goals

Marketplace listing beyond basic action metadata; signed containers; automated
release notes generation.

## Design References

DESIGN.md §18, §2.3 U6, §14.2 T10; ADR-003, ADR-007.
