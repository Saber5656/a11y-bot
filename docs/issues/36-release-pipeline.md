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
   - supply-chain hygiene (T10): every job uses `npm ci` against the committed
     lockfile; all third-party actions version-pinned;
   - jobs: verify (full CI suite reuse via `workflow_call` or duplicated steps
     — choose and document) → publish (`npm publish --provenance --access
     public`, plus `--tag next` when the version contains a prerelease suffix
     (`-`), `latest` otherwise; `NPM_TOKEN` from repo secret; registry
     2FA/automation-token setup documented for the maintainer, agent does not
     create tokens) → GitHub Release with changelog excerpt → move/force the
     `v1` major TAG to the release commit (documented action or script; this
     is the tag-ref exception already recorded in DESIGN §14.2 T6 — issue 35's
     threat-model doc must carry the same note).
3. Pre-publish assertions in the workflow: `npm pack` file list matches
   allowlist (dist, schemas, action.yml, README, LICENSE, CHANGELOG,
   package.json); bin executes and `node dist/cli/index.js --version` equals
   the tag **with the leading `v` stripped** (`v1.2.3` ↔ `1.2.3`); U6 name
   check: `npm view a11y-bot` must 404 (name free) — otherwise abort and
   switch to the documented fallback scope `@saber5656/a11y-bot` (GitHub
   owner's scope; README + action docs updated in the same change).
4. Release gate checklist at `docs/release/v1-gate-checklist.md` (executed for
   `1.0.0`, committed with checkboxes):
   - issues 01–35 all closed; this issue's own acceptance criteria green;
   - E2E green on the release commit;
   - security checklist from issue 35 all green;
   - scratch-repo manual validation of `fix-pr` + scheduled workflow (30/32)
     re-run on the release candidate;
   - README quickstart validated by following it verbatim on a fresh machine
     container (documented transcript).
5. `CHANGELOG.md`: keep-a-changelog format; the `1.0.0` entry is curated
   manually from ISSUE_PLAN.md's wave table + the closed-issue titles (no
   auto-generation dependency in v1).
6. README completion: badges (CI, npm), quickstart (init → scan → fix-pr),
   Action usage block, links to guides + security docs. README stays English;
   existing Japanese one-liner replaced (with the owner-approved English
   product statement + a short Japanese paragraph retained at the bottom for
   continuity — explicit owner sign-off on final README text before merge).

## Acceptance Criteria

- [ ] RC exercise green end-to-end (commands below): prerelease publish with
      provenance verified and `next` dist-tag — or the scoped fallback if U6
      fails, with docs updated accordingly.
- [ ] Pack-allowlist and version-match assertions demonstrated failing on a
      deliberate mismatch (negative test in a draft, reverted).
- [ ] `v1` major tag mechanism works (rc exercise) and the threat-model doc
      carries the T6 tag-exception note.
- [ ] T10 checks in release.yml verified: `npm ci` everywhere, pinned action
      versions (grep in workflow file).
- [ ] `docs/release/v1-gate-checklist.md` committed and fully checked for
      1.0.0.
- [ ] README final text owner-approved; quickstart transcript linked.

## Validation

```bash
npm pack --dry-run
# RC exercise (run on the release-candidate commit):
git tag v1.0.0-rc.1 && git push origin v1.0.0-rc.1
# wait for release.yml, then:
npm view a11y-bot@1.0.0-rc.1 dist-tags --json     # expect tag "next"
npm view a11y-bot@1.0.0-rc.1 --json | grep -i provenance
npm dist-tag ls a11y-bot                          # latest NOT moved by rc
git ls-remote origin refs/tags/v1                 # major tag points at rc commit
gh release view v1.0.0-rc.1 --json name,body      # GitHub Release created
```

## Dependencies

33, 34, 35 (33 is release-blocking: v1 ships with multi-provider support per
the ISSUE_PLAN completion statement).

## Non-goals

Marketplace listing beyond basic action metadata; signed containers; automated
release notes generation.

## Design References

DESIGN.md §18, §2.3 U6, §14.2 T10; ADR-003, ADR-007.
