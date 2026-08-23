# Plan 009: Make session invalidation epoch-aware so stale 401s cannot cause a login stampede

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/auth/session.ts src/client/dockhand-client.ts src/types/dockhand.ts tests/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P2
- **Effort**: S–M
- **Risk**: MED (touches the auth path every one of the 306 tools depends on)
- **Depends on**: none
- **Category**: security (availability / credential churn)
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

One `DockhandClient` — and therefore one `SessionManager` — is shared by every
MCP session in the process. When the Dockhand cookie expires, every in-flight
request fails with 401 at roughly the same time. Each 401 handler calls
`session.invalidate()` unconditionally, with no record of *which* cookie
produced that 401. So the sequence is: request 1's 401 invalidates cookie C1
and logs in for C2; requests 2..N's 401s — which were caused by the *already
dead* C1 — arrive later and each wipe the perfectly valid C2, triggering
another full login.

`login()`'s `loginPromise` only collapses logins that overlap **in time**, so a
burst spread across even a few hundred milliseconds produces a login stampede
against Dockhand: N logins where 1 was needed. Against a server with brute-force
protection — and this repo's README describes CrowdSec watching Dockhand 401s —
that is a self-inflicted lockout / rate-limit risk, and it amplifies one MCP
session's stale request into re-auth churn for every other session.

A secondary hazard sits in the same code: `getCookie()` ends with
`this.session!.cookie`, a non-null assertion that is only safe because nothing
calls `invalidate()` between `await this.login()` resolving and that line.
Nothing enforces that; making invalidation epoch-aware removes the hazard
rather than relying on microtask ordering.

## Current state

`src/auth/session.ts:12-35` — the manager and its login dedupe:

```typescript
export class SessionManager {
  private config: DockhandConfig;
  private session: SessionInfo | null = null;
  private loginPromise: Promise<void> | null = null;

  constructor(config: DockhandConfig) {
    this.config = config;
  }

  /**
   * Login to Dockhand and store the session cookie.
   */
  async login(): Promise<void> {
    // Prevent concurrent login attempts
    if (this.loginPromise) {
      return this.loginPromise;
    }

    this.loginPromise = this.performLogin();
    try {
      await this.loginPromise;
    } finally {
      this.loginPromise = null;
    }
  }
```

`src/auth/session.ts:170-185` — `getCookie` and `invalidate`:

```typescript
  async getCookie(): Promise<string> {
    if (!this.session || Date.now() >= this.session.expiresAt) {
      await this.login();
    }
    return this.session!.cookie;
  }

  /**
   * Invalidate current session (triggers re-login on next request).
   */
  invalidate(): void {
    this.session = null;
    log().info({ component: 'session' }, 'session invalidated, will re-login on next request');
  }
```

`src/client/dockhand-client.ts:435-447` — one of the four identical 401
handlers (`requestRaw`; the others are in `request`, `postSSE`, `putSSE` — find
them all with `grep -n "invalidate()" src/client/dockhand-client.ts`):

```typescript
    const headers: Record<string, string> = { 'Cookie': cookie, ...extraHeaders };

    let response = await this.loggedFetch(method, url, { method, headers, body });

    if (response.status === 401) {
      this.session.invalidate();
      // See postSSE() above: cancel this attempt's unread body so its #215
      // debug line still fires.
      await response.body?.cancel();
      const retryCookie = await this.session.getCookie();
      headers['Cookie'] = retryCookie;
      response = await this.loggedFetch(method, url, { method, headers, body });
    }
```

`SessionInfo` is declared in `src/types/dockhand.ts` — read it before
changing; it is one of only five interfaces in that file that are actually
used, so it is safe to extend.

Conventions:

- Session logging goes through `log()` from `src/utils/log-context.js` (not
  the bare `logger`) so lines carry `req`/`sid`/`call` when a request drives
  them.
- Existing session tests: `tests/session-login.test.ts`,
  `tests/session-login-debug-logging.test.ts`,
  `tests/login-failure-visibility.test.ts`. They stub `globalThis.fetch` with
  hand-rolled response objects — read `tests/session-login.test.ts` first and
  reuse its `mockResponse` shape.
- Note (context, not a task): several client tests replace the session with
  `{ getCookie: async () => 'session=x', invalidate: () => {} }`. If you change
  the `invalidate` signature, those stubs must still typecheck — see step 2.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Typecheck tests | `npm run typecheck:tests` | exit 0 |
| Session tests | `npx vitest run tests/session-login.test.ts tests/session-login-debug-logging.test.ts tests/login-failure-visibility.test.ts` | all pass |
| Client tests | `npx vitest run tests/client-debug-logging.test.ts tests/client-body-duration-215.test.ts tests/dockhand-client-error-redaction.test.ts tests/dockhand-client-delete-body.test.ts` | all pass |
| Full tests | `npm test` | all pass |

## Scope

**In scope**:
- `src/auth/session.ts` — add a generation counter; make `invalidate` take the
  generation it is invalidating; remove the `!` assertion.
- `src/types/dockhand.ts` — add `generation` to `SessionInfo` only.
- `src/client/dockhand-client.ts` — thread the generation through the four
  401 handlers.
- `tests/session-invalidation-race.test.ts` (create).
- Existing tests only where the signature change forces a mechanical update.

**Out of scope**:
- The retry *policy* (single retry per request) — do not add backoff, jitter,
  or a retry budget here; that is a separate design decision.
- The `logout` tool's interaction with the shared session
  (`src/tools/auth.ts`) — related and recorded as a follow-up, but it needs a
  product decision (is `logout` even meaningful on a shared client?).
- Making the client per-session instead of shared — architectural, recorded
  as a follow-up.
- `SESSION_TIMEOUT_MS` and the proactive-expiry check in `getCookie`.

## Git workflow

- Branch: `advisor/009-session-epoch`.
- Commits (scopes from `.commitlintrc.json`):
  - `fix(auth): make session invalidation epoch-aware to stop login stampedes`
  - `test(tests): cover concurrent stale-401 invalidation`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Give each session a generation

In `src/types/dockhand.ts`, add a required `generation: number` field to
`SessionInfo`.

In `src/auth/session.ts`:

- Add a private counter: `private generation = 0;`
- Wherever `performLogin()` assigns `this.session = { ... }`, increment first
  and stamp the value: `this.session = { ...existingFields, generation: ++this.generation };`
  (read the assignment site — it is inside `performLogin`, after the cookie is
  extracted).
- Change `getCookie()` to return both values:

```typescript
  /**
   * Get the current session cookie plus the generation it belongs to. The
   * generation lets a caller that later receives a 401 prove the cookie it
   * used is still the current one before invalidating — without that, a 401
   * caused by an already-replaced cookie wipes a freshly obtained session and
   * triggers a redundant login (one per concurrent stale request).
   */
  async getCookie(): Promise<{ cookie: string; generation: number }> {
    if (!this.session || Date.now() >= this.session.expiresAt) {
      await this.login();
    }
    const session = this.session;
    if (!session) {
      throw new Error('Dockhand login succeeded but no session was stored');
    }
    return { cookie: session.cookie, generation: session.generation };
  }
```

- Change `invalidate` to be generation-scoped:

```typescript
  /**
   * Invalidate the session identified by `generation` (triggers re-login on
   * the next request). A no-op when the current session is already newer —
   * that means another caller re-logged in between this request being sent
   * and its 401 arriving, and the newer cookie must survive.
   */
  invalidate(generation: number): void {
    if (this.session && this.session.generation !== generation) {
      log().debug(
        { component: 'session', staleGeneration: generation, currentGeneration: this.session.generation },
        'ignoring 401 for a superseded session',
      );
      return;
    }
    this.session = null;
    log().info({ component: 'session' }, 'session invalidated, will re-login on next request');
  }
```

**Verify**: `npm run typecheck` → it will now report errors at every
`getCookie()`/`invalidate()` call site. That is expected; step 2 fixes them.

### Step 2: Thread the generation through the client's 401 handlers

In `src/client/dockhand-client.ts`, update **every** call site — find them
with `grep -n "getCookie()\|invalidate()" src/client/dockhand-client.ts`
(expect four 401 blocks plus the initial cookie fetch in each request method).
The pattern per site:

```typescript
    const { cookie, generation } = await this.session.getCookie();
    const headers: Record<string, string> = { 'Cookie': cookie, ...extraHeaders };

    let response = await this.loggedFetch(method, url, { method, headers, body });

    if (response.status === 401) {
      this.session.invalidate(generation);
      await response.body?.cancel();
      const retry = await this.session.getCookie();
      headers['Cookie'] = retry.cookie;
      response = await this.loggedFetch(method, url, { method, headers, body });
    }
```

Preserve every existing comment in those blocks (the `#215` body-cancel notes
are load-bearing documentation).

Then check the test stubs: tests that substitute a fake session object
(`{ getCookie: async () => 'session=x', invalidate: () => {} }`) will fail
`npm run typecheck:tests`. Update those stubs mechanically to
`{ getCookie: async () => ({ cookie: 'session=x', generation: 1 }), invalidate: () => {} }`.
Do not change what those tests assert.

**Verify**: `npm run typecheck` and `npm run typecheck:tests` → exit 0;
then the Session tests and Client tests commands from the table → all pass.

### Step 3: Regression test for the stampede

Create `tests/session-invalidation-race.test.ts`. Pattern the fetch stubbing
on `tests/session-login.test.ts`.

Build a `SessionManager` against a scripted `globalThis.fetch` that:
- answers `POST /api/auth/login` with a 200 + `set-cookie`, using a different
  cookie value per call, and counts the calls;
- lets you drive the ordering deliberately.

Cases:

1. **Stale 401 does not wipe a fresh session.** Call `getCookie()` → `{cookie: C1, generation: g1}`.
   Call `invalidate(g1)` (simulating request 1's 401) and `getCookie()` again
   → `{cookie: C2, generation: g2}` with `g2 !== g1`. Now call
   `invalidate(g1)` again (request 2's *stale* 401). Assert the next
   `getCookie()` returns **C2 without a further login** — i.e. the login call
   count is exactly 2, not 3.
2. **Current-generation invalidate still works.** `invalidate(g2)` → the next
   `getCookie()` performs a login (count becomes 3) and returns a new cookie.
3. **Concurrent stale 401s.** Simulate N=5 callers that all captured `g1`,
   then have all five call `invalidate(g1)` after a re-login has produced
   `g2`. Assert the login count stayed at 2.
4. **`getCookie` throws a typed error, not a `TypeError`,** if login resolves
   without storing a session — force it by stubbing login to resolve while
   leaving `session` null (e.g. spy on `performLogin`), and assert the thrown
   message matches the step-1 text.

**Verify**: `npx vitest run tests/session-invalidation-race.test.ts` → all
pass; `npm test` → all pass.

## Test plan

Four cases in `tests/session-invalidation-race.test.ts` (above), patterned on
`tests/session-login.test.ts`. Existing session/client suites must stay green
with only mechanical stub updates — if any of them needs an *assertion*
changed, that is a STOP condition.

## Done criteria

- [ ] `npm run typecheck` and `npm run typecheck:tests` exit 0
- [ ] `npm test` exits 0 including 4 new tests
- [ ] `grep -n "invalidate()" src/ ` → no matches (every call passes a generation)
- [ ] `grep -n "this.session!" src/auth/session.ts` → no matches
- [ ] `grep -c "getCookie()" src/client/dockhand-client.ts` matches the number
      of destructured call sites (no site left on the old string-returning API)
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- The excerpts no longer match (drift) — especially if the 401 handling has
  been refactored into a shared helper since `3d55424` (that would be a
  better place for this change; report and propose it).
- An existing test's **assertion** (not just its stub shape) must change to
  pass — report which and why before proceeding.
- You find a fifth `invalidate()` call site outside `src/client/dockhand-client.ts`
  (e.g. in a tool) — report it; it may need the same treatment or may be a
  deliberate unconditional invalidation.
- Making `getCookie` return an object breaks a public API you cannot see from
  here (`grep -rn "getCookie" src/ tests/` first) — report the full call-site
  list before changing the signature.

## Maintenance notes

- The generation counter is per-`SessionManager` instance and never persists;
  it only needs to be monotonic within a process lifetime.
- If the client is ever made per-MCP-session (a recorded follow-up), this fix
  stays correct but its urgency drops — the stampede is a consequence of
  sharing one credential across sessions.
- Related, deliberately deferred: the `logout` tool
  (`src/tools/auth.ts`) destroys the shared upstream cookie for every session
  without invalidating the local `SessionManager`, so callers self-heal only
  via a wasted 401 round trip. Decide whether that tool should invalidate
  locally or be removed.
- Reviewer should scrutinize: that all four 401 blocks were updated
  identically, that the `#215` body-cancel calls survived, and that no path
  can now loop on repeated 401s (the retry is still exactly one attempt).
