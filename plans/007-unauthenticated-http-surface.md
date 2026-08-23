# Plan 007: Stop the unauthenticated HTTP surface from leaking stack traces and deployment detail

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/server.ts tests/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none (independent of plan 001, which touches a different
  part of `src/server.ts` — the POST handler; coordinate if both are in
  flight)
- **Category**: security
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

Two disclosures on the parts of the HTTP surface that no guard protects:

1. **Stack traces to unauthenticated callers.** `express.json()` is registered
   *before* the transport guards — deliberately, so rejected requests still
   produce an access line for CrowdSec. A malformed or oversized body
   therefore reaches Express's default final handler *without ever meeting the
   bearer guard*, and `finalhandler` serialises `err.stack` into the response
   body for any `NODE_ENV` other than `production`. The mitigation currently
   lives **only in the Dockerfile** (`ENV NODE_ENV=production`). The
   README documents "From Source" (`npm start`) and `npm run dev` as supported
   paths, and neither sets `NODE_ENV` — so those deployments hand container
   paths and dependency line offsets to any unauthenticated caller. The
   Dockerfile comment states the team already knows this failure mode; the fix
   is to stop depending on an env var for it.

2. **`/health` describes the deployment to anyone.** It is registered before
   the guards (always unauthenticated) and returns the exact version plus live
   session counts and configured TTLs — a version to match against a changelog
   and a readout of how close the process is to its session ceiling.

## Current state

`src/server.ts` — middleware order (the comment above `express.json()`
explains why it must stay first; do not reorder it):

```typescript
  app.use(createAccessLogMiddleware(trustedProxies));
  // And ahead of express.json() for the same reason, which is less obvious: a body
  // parser rejects a malformed or oversized payload by calling next(err), and that
  // skips every remaining non-error middleware. Registered the other way round, this
  // middleware never runs for such a request at all — so it never attaches its
  // res.on('finish') handler, and a 400 or a 413 produces no access line whatsoever.
  // Malformed-payload probing is exactly what CrowdSec is here to see.
  app.use(express.json());
```

`/health`, registered at `src/server.ts:97-111` — i.e. **before** the guards
at lines 195-197:

```typescript
  app.get('/health', (_req: Request, res: Response) => {
    res.json({
      status: 'ok',
      server: 'mcp-dockhand',
      version: pkg.version,
      sessions: {
        active: sessions.size,
        pending: pendingSessions,
        max: lifecycle.maxSessions === 0 ? null : lifecycle.maxSessions,
        ttlSeconds: lifecycle.inactivityTimeoutMs / 1000,
        cleanupIntervalSeconds: lifecycle.cleanupIntervalMs / 1000,
      },
    });
  });
```

The guards, applied to `/mcp` only:

```typescript
    app.use('/mcp', createHostOriginGuard(security.allowedHosts, security.allowedOrigins));
  }
  app.use('/mcp', createBearerAuthGuard(security.authToken));
```

Verified facts you can rely on:

- `node_modules/finalhandler/index.js` `getErrorMessage(err, status, env)`:
  `if (env !== 'production') { msg = err.stack }`.
- `grep -rn "NODE_ENV" src/ Dockerfile package.json` → only `Dockerfile:25`
  sets it (plus `src/utils/logger.ts` reading `NODE_ENV === 'test'` for its
  sync-destination choice — unrelated, do not touch).
- There is **no** `ErrorRequestHandler` registered anywhere in `src/`
  (`grep -n "app.use" src/server.ts` shows only the four middleware lines).
- `src/auth/transport-guard.ts` exports the guard factories and
  `getTransportSecurityConfig(env)`; the bearer comparison helper
  `timingSafeTokenMatch` is module-private.

Conventions:

- Request-context logging uses `log()` from `src/utils/log-context.js`, not
  the bare `logger`.
- The healthcheck in `Dockerfile` and `docker-compose.yml` calls
  `wget -qO- http://127.0.0.1:8080/health` and only checks the exit code —
  `tests/healthcheck-loopback.test.ts` regex-asserts that text, so the
  endpoint must keep returning 200 with a JSON body.
- e2e tests boot the real app on a fixed loopback port — see
  `tests/transport-security-e2e.test.ts` (ports in use: 48213, 48299, 48301).

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Typecheck tests | `npm run typecheck:tests` | exit 0 |
| Full tests | `npm test` | all pass |
| One file | `npx vitest run tests/<file>` | all pass |

## Scope

**In scope**:
- `src/server.ts` — add a terminal error handler; reduce the default
  `/health` payload.
- `tests/unauthenticated-surface.test.ts` (create).
- `tests/healthcheck-loopback.test.ts` — only if its assertions break.

**Out of scope**:
- The middleware ORDER (`accessLog` → `express.json()` → guards). It is
  deliberate and documented; the error handler goes at the END, not by
  reordering these.
- `Dockerfile`'s `ENV NODE_ENV=production` — keep it; this plan removes the
  *dependence* on it, it does not remove the setting.
- The POST `/mcp` handler internals — plan 001 owns those lines.
- `MCP_MAX_SESSIONS`'s unlimited default — a separate product decision
  recorded in `plans/README.md`.
- `src/auth/transport-guard.ts` — do not change guard semantics.

## Git workflow

- Branch: `advisor/007-unauth-surface`.
- Commits (scopes from the closed list in `.commitlintrc.json`):
  - `fix(security): return a fixed error body instead of Express stack traces`
  - `fix(security): reduce the unauthenticated /health payload`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Add a terminal error handler

At the END of `createServer`'s middleware/route registration in
`src/server.ts` — after all routes are registered, immediately before the
function returns the app/starts listening — add an Express 5
`ErrorRequestHandler` (four parameters; the arity is what makes Express treat
it as an error handler):

```typescript
  // Terminal error handler. express.json() runs ahead of the transport guards
  // (see the comment on its registration above), so a malformed or oversized
  // body from an UNAUTHENTICATED caller reaches Express's default final
  // handler — which serialises err.stack into the response body for any
  // NODE_ENV other than 'production'. Depending on an env var for that is how
  // a from-source deployment (`npm start`, `npm run dev` — neither sets
  // NODE_ENV) leaks container paths to anyone who can reach the port. Answer
  // with a fixed body instead, and keep the detail server-side.
  app.use((err: unknown, _req: Request, res: Response, _next: NextFunction) => {
    const status = typeof (err as { status?: unknown })?.status === 'number'
      ? (err as { status: number }).status
      : 500;
    log().warn({ component: 'server', err, status }, 'request rejected before routing');
    if (!res.headersSent) {
      res.status(status).json({ error: status === 500 ? 'Internal server error' : 'Bad request' });
    }
  });
```

Import `NextFunction` from `express` alongside the existing `Request`/`Response`
type imports. Keep the `status` passthrough: `express.json()` sets 400 for
malformed JSON and 413 for oversized bodies, and losing those would change
what CrowdSec sees in the access log.

**Verify**: `npm run typecheck` → exit 0; `npm test` → all pass.

### Step 2: Reduce the default `/health` payload

Change the handler so the always-public response is minimal, and the detailed
payload is returned only to a caller that presents the configured bearer
token. When `MCP_AUTH_TOKEN` is unset there is no way to authenticate, so the
detail is simply not served — that is the intended outcome, and it does not
break the container healthcheck (which only checks the exit code).

Target shape:

```typescript
  app.get('/health', (req: Request, res: Response) => {
    const base = { status: 'ok', server: 'mcp-dockhand' };
    // Version and live session accounting describe the deployment (a version
    // to match against a changelog; how close the process is to its session
    // ceiling). /health is registered ahead of the transport guards and is
    // therefore always reachable unauthenticated — so serve the detail only
    // to a caller holding the configured bearer token.
    if (!security.authToken || !hasValidBearer(req, security.authToken)) {
      res.json(base);
      return;
    }
    res.json({
      ...base,
      version: pkg.version,
      sessions: {
        active: sessions.size,
        pending: pendingSessions,
        max: lifecycle.maxSessions === 0 ? null : lifecycle.maxSessions,
        ttlSeconds: lifecycle.inactivityTimeoutMs / 1000,
        cleanupIntervalSeconds: lifecycle.cleanupIntervalMs / 1000,
      },
    });
  });
```

`hasValidBearer` does not exist yet. Add it to
`src/auth/transport-guard.ts` as a small exported helper that reuses the
existing private `timingSafeTokenMatch` and `BEARER_PREFIX` — read the file
first; the bearer guard already contains exactly this parsing logic, so
factor that shared piece out rather than writing a second parser. Import it in
`src/server.ts`.

**Verify**: `npm run typecheck` → exit 0;
`npx vitest run tests/healthcheck-loopback.test.ts tests/transport-security-e2e.test.ts` → all pass.

### Step 3: Tests

Create `tests/unauthenticated-surface.test.ts`, modeled on
`tests/transport-security-e2e.test.ts` (boot the real server on a fresh fixed
port — use 48311 — with dummy `DOCKHAND_*` env vars; close it in `afterAll`).

Cases:

1. **No stack trace on a malformed body, unauthenticated.** POST `/mcp` with
   `content-type: application/json` and a body of `{"broken"` (invalid JSON).
   Assert: status is 400, and the response text contains neither `"stack"`
   nor the string `at ` followed by a file path — the strongest simple
   assertion is that the parsed body deep-equals `{ error: 'Bad request' }`.
   Run this test **without** `NODE_ENV=production` in the test env so it
   actually exercises the previously-leaking path (vitest sets
   `NODE_ENV=test`; confirm the test process is not production, otherwise the
   test proves nothing — assert `process.env.NODE_ENV !== 'production'` at the
   top of the case).
2. **`/health` unauthenticated is minimal** when `MCP_AUTH_TOKEN` is set:
   body deep-equals `{ status: 'ok', server: 'mcp-dockhand' }`; no `version`
   key, no `sessions` key.
3. **`/health` with the correct bearer** returns `version` and `sessions`.
4. **`/health` with `MCP_AUTH_TOKEN` unset** returns the minimal body and
   still 200 (healthcheck compatibility).

**Verify**: `npx vitest run tests/unauthenticated-surface.test.ts` → all pass;
`npm test` → all pass.

## Test plan

Four cases in `tests/unauthenticated-surface.test.ts` as above, patterned on
`tests/transport-security-e2e.test.ts`. No existing test should need
weakening; if `tests/healthcheck-loopback.test.ts` fails, it asserts on
Dockerfile/compose *text* (not the payload) — re-read it before touching it,
and report if the failure is not obviously caused by this change.

## Done criteria

- [ ] `npm run typecheck` and `npm run typecheck:tests` exit 0
- [ ] `npm test` exits 0 including 4 new tests
- [ ] `grep -n "NextFunction" src/server.ts` shows the error handler exists
- [ ] An unauthenticated malformed-body POST returns a fixed JSON error with
      no `stack` (covered by test 1)
- [ ] `/health` unauthenticated exposes neither `version` nor `sessions`
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- The middleware block or `/health` handler no longer matches the excerpts
  (drift) — in particular if plan 001 has already landed and reshaped the
  POST handler, re-read `src/server.ts` before editing.
- Adding the error handler breaks any access-log ordering test
  (`tests/access-log-ordering-e2e.test.ts`,
  `tests/access-log-middleware.test.ts`) — the access line must still be
  written on `finish`. Report the failing assertion rather than reordering
  middleware.
- `timingSafeTokenMatch` cannot be reused without changing guard behavior —
  report instead of duplicating the comparison logic.
- Any operator-facing doc or dashboard in the repo depends on the public
  `/health` detail (grep `README.md` and `docs/` for `"/health"`) — if the
  README documents the full payload as a monitoring contract, report before
  changing it, and update the doc in the same commit if you proceed.

## Maintenance notes

- If a future change adds a route outside `/mcp`, remember the guards are
  path-scoped to `/mcp` — new routes are unauthenticated by default. The
  error handler added here is global and will cover them.
- Reviewer: confirm the error handler is registered **last** (Express matches
  in registration order) and that its arity is exactly 4.
- Deferred, recorded in `plans/README.md`: a finite `MCP_MAX_SESSIONS`
  default (bounding unauthenticated session allocation) — plan 001 removes
  the leak that made it urgent, but the cap is still worth a decision.
