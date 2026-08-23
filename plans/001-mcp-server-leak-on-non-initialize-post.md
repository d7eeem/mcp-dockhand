# Plan 001: Stop leaking McpServer instances on non-initialize POSTs and transport self-close

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/server.ts src/session-lifecycle.ts tests/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: bug (security-adjacent: unauthenticated memory exhaustion)
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

Every `POST /mcp` without an `Mcp-Session-Id` header builds a full `McpServer`
(306 tool registrations) and connects it to a fresh transport **before** the
SDK decides whether the request is actually an `initialize`. When the SDK
rejects the request (non-initialize without session header, or unsupported
protocol version), it **resolves** with a 400/404 JSON response — it does not
throw. The handler's `catch` block therefore never runs, `onsessioninitialized`
never fires, and the connected `McpServer` + transport are neither stored in
the `sessions` map nor closed: one permanent leak per malformed request. With
the documented default of no `MCP_AUTH_TOKEN`, this is reachable
unauthenticated and each leaked server retains several MB of tool
registrations. A second, smaller asymmetry: when a transport closes on its own,
`transport.onclose` deletes the map entry but never calls `server.close()`,
unlike every other removal path.

## Current state

Relevant files:

- `src/server.ts` — Express app + `/mcp` route handlers; contains both defects.
- `src/session-lifecycle.ts` — pure lifecycle helpers; `removeSessionEntry()`
  (lines 146–164) is the exemplar for "close a session properly". Do not
  modify this file.
- `node_modules/@modelcontextprotocol/sdk/dist/esm/server/webStandardStreamableHttp.js`
  — `validateSession()` returns (not throws) `createJsonErrorResponse(400, ...)`
  for a non-initialize POST without a session id, and
  `validateProtocolVersion()` does the same for unsupported versions. This is
  why the leak path is a *successful* `handleRequest` resolution.

The POST handler today (`src/server.ts:219-283`, abridged):

```typescript
      let initializedSessionId: string | undefined;
      let server: McpServer | undefined;
      let transport: StreamableHTTPServerTransport | undefined;
      try {
        transport = new StreamableHTTPServerTransport({
          sessionIdGenerator: () => crypto.randomUUID(),
          // ...
          onsessioninitialized: (id) => {
            initializedSessionId = id;
            beginFoundingSession(sessions, id, { server: server!, transport: transport! });
            // ...
          },
        });

        transport.onclose = () => {
          const sid = [...sessions.entries()].find(([, entry]) => entry.transport === transport)?.[0];
          if (sid) {
            sessions.delete(sid);
            log().info({ component: 'session', sid, active: sessions.size }, 'session transport closed');
          }
        };

        server = createMcpServer(client);
        await server.connect(transport);
        await transport.handleRequest(req, res, req.body);

        if (initializedSessionId) {
          completeFoundingSession(sessions, initializedSessionId);
        }
      } catch (error) {
        if (initializedSessionId) {
          await removeSession(initializedSessionId, 'initialization failure');
        } else if (server) {
          try {
            await server.close();
          } catch { /* Best-effort cleanup for a failed initialization. */ }
        }
        throw error;
      } finally {
        releasePendingSessionSlot();
      }
```

Note the gap: on the **success** path (`handleRequest` resolved, SDK answered
400/404 itself), if `initializedSessionId` is still `undefined`, nothing closes
`server`.

The other removal paths all close the server — see
`src/session-lifecycle.ts:146-164` (`removeSessionEntry`): `sessions.delete`
first, then `await entry.server.close()` with a `transport.close?.()` fallback,
both wrapped so a racing close cannot throw out. `transport.onclose` at
`src/server.ts:252-258` is the one path that skips `server.close()`.

Repo conventions that apply:

- Logging inside a request context uses `log()` from
  `src/utils/log-context.js`, not the bare `logger` — see the header comment
  of `src/session-lifecycle.ts:1-4`.
- The SDK chains `onclose` handlers set before `server.connect()`
  (`shared/protocol.js` keeps the previous handler), so the existing
  `transport.onclose` assignment position must not move below `connect`.
- Strict TypeScript; no `any` unless the surrounding code already uses it.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Typecheck tests | `npm run typecheck:tests` | exit 0 |
| Full tests | `npm test` | all pass (~1120 tests) |
| One file | `npx vitest run tests/<file>` | all pass |

(Lint requires Docker — `npm run lint` runs `scripts/lint-in-container.sh`.
Run it if Docker is available; skip with a note otherwise.)

## Scope

**In scope** (the only files you should modify):
- `src/server.ts`
- `tests/session-leak-on-bad-post.test.ts` (create)

**Out of scope** (do NOT touch, even though they look related):
- `src/session-lifecycle.ts` — the helpers are correct; the bug is in wiring.
- `MCP_MAX_SESSIONS` default in `src/session-lifecycle.ts:36` — changing the
  unlimited default is a separate product decision (recorded in
  `plans/README.md` as a considered follow-up).
- The SDK's own 400/404 bodies — do not try to pre-parse the JSON-RPC body to
  short-circuit before constructing the transport. It is a valid alternative
  design, but it duplicates SDK validation logic; this plan takes the
  minimal-close approach instead.

## Git workflow

- Branch: `advisor/001-mcp-server-leak` off the current branch.
- Conventional Commits with a **mandatory scope from the closed list** in
  `.commitlintrc.json` (`scope-empty: never`). Use: `fix(security): close
  McpServer built for requests that never initialize a session`.
  Valid scopes include: `security`, `tests`, `system`, `tools`, `client`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Close the server when no session was initialized (success path)

In `src/server.ts`, after `await transport.handleRequest(req, res, req.body);`
and the existing `if (initializedSessionId) { completeFoundingSession(...) }`
block, add the cleanup for the not-initialized case:

```typescript
        if (initializedSessionId) {
          completeFoundingSession(sessions, initializedSessionId);
        } else {
          // The SDK resolved without initializing a session (it answered the
          // request itself with a 4xx: non-initialize POST without a session
          // id, or unsupported protocol version). Nothing owns this server —
          // close it or it leaks with all 306 tool registrations attached.
          await server.close().catch(() => {});
        }
```

`server` is definitely assigned at this point (the `await`s above it would
have thrown otherwise), so no optional chaining is needed — but keep
TypeScript happy: `server` is typed `McpServer | undefined`; either narrow via
the surrounding control flow or use `server?.close()`. Note `server.close()`
also closes the connected transport (SDK behavior), which is what we want.

**Verify**: `npm run typecheck` → exit 0.

### Step 2: Close the server in `transport.onclose`

Replace the `transport.onclose` body (`src/server.ts:252-258`) so the entry's
server is closed too, with a re-entrancy guard (closing the server triggers
the transport close chain again):

```typescript
        let selfCloseHandled = false;
        transport.onclose = () => {
          if (selfCloseHandled) return;
          selfCloseHandled = true;
          const found = [...sessions.entries()].find(([, entry]) => entry.transport === transport);
          if (found) {
            const [sid, entry] = found;
            sessions.delete(sid);
            void entry.server.close().catch(() => {});
            log().info({ component: 'session', sid, active: sessions.size }, 'session transport closed');
          }
        };
```

Keep the assignment exactly where it is today (before `server.connect(transport)`),
because the SDK chains the pre-connect handler.

**Verify**: `npm run typecheck` → exit 0, and `npm test` → all pass
(`tests/founding-session-correlation.test.ts` and
`tests/transport-security-e2e.test.ts` exercise adjacent behavior and must
stay green).

### Step 3: Add the regression test

Create `tests/session-leak-on-bad-post.test.ts`, modeled structurally on
`tests/transport-security-e2e.test.ts` (real `createServer` on a loopback
port, real HTTP requests, teardown in `afterAll`). Use a distinct fixed port
not used by other e2e files (existing ones use 48213, 48299, 48301 — pick
e.g. 48307).

Test cases:

1. **Non-initialize POST without session id gets a 4xx and leaks nothing.**
   Send `POST /mcp` with a JSON-RPC body that is *not* an initialize request,
   e.g. `{"jsonrpc":"2.0","id":1,"method":"tools/list"}`, headers
   `content-type: application/json` and `accept: application/json, text/event-stream`.
   Expect a 400 response. Then send a valid `initialize` request and confirm
   it still succeeds (server remains functional).
2. **Leak assertion.** The `sessions` map is private to `createServer`, so
   assert indirectly: after N (e.g. 5) bad POSTs, `GET /health` must report
   `sessions.active: 0` (the `/health` payload includes session counts — see
   `src/server.ts:98-111`). Before the fix, active stays 0 too (the leak is
   *outside* the map), so additionally assert on the observable that does
   change: spy on the log output for `'session transport closed'` /
   absence of `'session created'`, or — simpler and sufficient — assert that
   repeated bad POSTs each return 400 and that a subsequent initialize
   succeeds, and rely on the code-level close (step 1) being covered by the
   review. If you can cheaply observe closed state (e.g. by counting
   `registerAllTools` log lines `'all Dockhand tools registered'` via the
   `fs.writeSync` capture pattern from `tests/session-lifecycle.test.ts:16-27`),
   prefer that: bad POSTs still build a server today (registration line
   fires), but after the fix the built server must be closed — the assertion
   is that the process does not accumulate open transports; at minimum assert
   the 400 + still-functional behavior and document the limitation in a
   comment.

**Verify**: `npx vitest run tests/session-leak-on-bad-post.test.ts` → all pass.

## Test plan

- New: `tests/session-leak-on-bad-post.test.ts` (cases above).
- Pattern: `tests/transport-security-e2e.test.ts` for the server-boot harness;
  `tests/session-lifecycle.test.ts:16-27` for the `fs.writeSync` logger
  capture helper.
- Verification: `npm test` → all pass including the new file.

## Done criteria

Machine-checkable. ALL must hold:

- [ ] `npm run typecheck` and `npm run typecheck:tests` exit 0
- [ ] `npm test` exits 0; `tests/session-leak-on-bad-post.test.ts` exists and passes
- [ ] `grep -n "server.close" src/server.ts` shows a close call on the
      non-initialized success path AND inside `transport.onclose`
- [ ] No files outside the in-scope list are modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

Stop and report back (do not improvise) if:

- The code at `src/server.ts:219-283` no longer matches the excerpt (drift).
- `server.close()` inside `onclose` causes a test hang or infinite recursion
  that the `selfCloseHandled` guard does not stop — report the observed call
  chain instead of adding more guards.
- The SDK version has changed from `@modelcontextprotocol/sdk` 1.30.x
  (check `package-lock.json`) — the "400 resolves, not throws" premise must be
  re-verified against the new version's `webStandardStreamableHttp.js` first.
- Any existing test fails after step 2 in a way that is not obviously an
  assertion on the old (buggy) behavior.

## Maintenance notes

- If a future change adds pre-parsing of the JSON-RPC body to reject
  non-initialize requests before constructing the transport (the cheaper
  design), the step-1 close becomes dead code — remove it then, not before.
- Reviewer should scrutinize: double-close safety (step 1 close vs. the
  `catch` block close vs. `onclose`) — `McpServer.close()` must be idempotent
  in practice; the `.catch(() => {})` guards are load-bearing.
- Deferred (recorded in plans/README.md): finite `MCP_MAX_SESSIONS` default
  and trimming the unauthenticated `/health` payload.
