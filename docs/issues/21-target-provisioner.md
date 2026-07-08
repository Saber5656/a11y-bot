# Title

Audit target provisioning: url / staticDir / command variants

## Summary

Implement the lifecycle that turns each `audit.targets[]` entry into a reachable
base URL for the browser session, with readiness checks and guaranteed teardown.

## Context

DESIGN.md §11.1 defines the state machine and variant table. Config validation
already exists (issue 02); this issue owns runtime behavior. Trust model: target
config including `command` is trusted repo input (§14.1) — documented, not
sandboxed.

## Scope

- `src/audit/provision.ts` (+ static file server helper).

## Detailed Requirements

1. API: `provisionTargets(config, logger): Promise<ProvisionedTarget[]>` where
   `ProvisionedTarget = { name, baseUrl, teardown(): Promise<void>,
   state: TargetState }` and `TargetState = "PENDING" | "PROVISIONING" |
   "READY" | "AUDITING" | "DONE" | "FAILED" | "TORN_DOWN"` — exactly the
   DESIGN §11.1 machine; the provisioner sets PENDING→PROVISIONING→READY|FAILED
   and TORN_DOWN; the audit command (27) advances READY→AUDITING→DONE.
   Plus `teardownAll()` that never throws (logs failures).
2. `url` variant: readiness = GET (redirects followed ≤ 5, HTTPS/HTTP only —
   other schemes → ConfigError at load already; double-check here) returning
   **2xx/3xx** within the variant's `readyTimeoutMs` (schema default 60000,
   DESIGN §6.3; poll 1 s). No teardown.
3. `staticDir` variant:
   - in-process HTTP server bound to `127.0.0.1:0` (ephemeral port), serving the
     directory resolved against repo root; refuse (ConfigError) if the resolved
     path escapes the repo root (path traversal guard, §14.2 T11);
   - content types via a minimal extension map; directory index `index.html`;
     `spaFallback: true` → unknown non-asset paths return `/index.html`;
   - no directory listings; `..` segments rejected with 403 (test);
   - teardown closes the server (await close).
4. `command` variant:
   - spawn with `shell: true`, cwd repo root, env passthrough plus
     `PORT=<port>`; capture stdout/stderr to debug log ring buffer (last 100
     lines shown on failure);
   - readiness: TCP connect then GET `readyPath` on `127.0.0.1:<port>` until
     `readyTimeoutMs`; process exit before ready → FAILED with captured output;
   - teardown: SIGTERM to the **process group** (spawn with `detached: true`),
     SIGKILL after 10 s; reap zombies; idempotent.
5. Failure policy: per-target FAILED recorded with reason; all targets failed →
   the audit command exits 4 (wired in 27; here return states). Teardown always
   runs (try/finally), including on SIGINT (install once, restore after).
6. Port allocation: OS-assigned (`listen(0)`) for staticDir; `command` uses the
   configured port — if occupied, fail fast with a clear EnvError (hint: port
   in use) rather than auditing someone else's server (correctness+safety).

## Acceptance Criteria

- [ ] All three variants provision and teardown in tests (command variant uses a
      tiny node http script fixture).
- [ ] Path traversal (`staticDir: ../..` and URL `..` requests) rejected (tests).
- [ ] SPA fallback semantics tested (asset 404 vs route fallback).
- [ ] Occupied-port failure for command variant tested.
- [ ] Ready-timeout produces FAILED with last-output excerpt; zombie-free
      (`ps` check in test on POSIX).
- [ ] Security (DESIGN §14.2 T3): captured command stdout/stderr passes through
      the issue-04 logger redaction — a fixture command echoing a planted
      `GITHUB_TOKEN` value shows `***` in the failure excerpt.
- [ ] `url` readiness: 404 response does NOT become READY (2xx/3xx only).
- [ ] SIGINT during provisioning tears down children (manual test documented +
      automated where feasible).

## Validation

```bash
npm test -- src/audit/provision
```

## Dependencies

02, 04.

## Non-goals

Browser work (22+), crawling, HTTPS termination for local servers, auth flows.

## Design References

DESIGN.md §11.1, §14.1 (trust), §14.2 T9/T11, §12.3 (exit 4).
