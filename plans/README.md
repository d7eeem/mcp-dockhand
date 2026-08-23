# Implementation Plans

Advisor plans for mcp-dockhand, written against commit `3d55424` on branch
`hardened`. Execute in the order below unless dependencies say otherwise. Each
executor: read the plan fully before starting, honor its STOP conditions, and
update your row when done.

Two runs are recorded here:

- **2026-08-20, deep audit (all nine categories)** → plans 001–006.
- **2026-08-20, security-focused audit** → plans 007–013. This run verified
  the security findings the deep run left unplanned, and added one lens the
  deep pass under-covered: the server as a *confused deputy* (tool-level blast
  radius, and third-party-controlled content flowing back into the model).

> Note: this directory is the advisor-plan index and is unrelated to
> `docs/plans/`, which is the repo's historical feature-plan archive.
>
> Selection note: both runs were non-interactive, so plans were written for
> the top findings by leverage (impact ÷ effort, weighted by confidence). The
> larger candidates under "Follow-ups awaiting a maintainer decision" need a
> product or scope call before they can be planned honestly.

## Execution order & status

| Plan | Title | Priority | Effort | Depends on | Status |
|------|-------|----------|--------|------------|--------|
| 001 | Stop leaking McpServer on non-initialize POSTs / transport self-close | P1 | S | — | TODO |
| 002 | Fix export_image/export_volume binary corruption + raw download cap | P1 | S | — | TODO |
| 003 | Reject newlines in stack .env keys/values | P1 | S | — | TODO |
| 004 | Compose env pass-through, .env.example placeholders, ci.yml permissions, release npm ci | P1 | S–M | — | TODO |
| 007 | Terminal error handler (no stack traces) + minimize unauthenticated /health | P1 | S | — | TODO |
| 008 | Jenkins fork-sync: gate before executing upstream code, shrink privileges | P1 | M | — | TODO |
| 011 | Mark third-party-controlled tool output as untrusted data | P1 | M | 006 (soft) | TODO |
| 005 | Correctness sweep (7 small verified bugs) | P2 | M | — | TODO |
| 006 | Compact JSON tool responses | P2 | S | — | TODO |
| 009 | Epoch-aware session invalidation (stop login stampedes) | P2 | S–M | — | TODO |
| 010 | CI supply chain: narrow Infisical pull, SHA-pin actions | P2 | S–M | 004 | TODO |
| 012 | Credential-suffix parity + one-time-credential warnings | P2 | S | — | TODO |
| 013 | Flag-gate host-escape capabilities (privileged settings, host files) | P2 | M | — | TODO |

Status values: TODO | IN PROGRESS | DONE | BLOCKED (with one-line reason) |
REJECTED (with one-line rationale).

## Dependency notes

- **010 after 004** — both edit workflow files; 004 adds `ci.yml`'s
  `permissions:` block and fixes `release.yml`'s install. Avoid conflicts.
- **011 after 006** (soft) — both touch `src/utils/response.ts`. Either order
  works; 006 first avoids a merge conflict in the same three helpers.
- **007 vs 001** — both edit `src/server.ts`, but different regions (007: the
  middleware tail and `/health`; 001: the POST handler). Coordinate if both
  are in flight simultaneously.
- **012 and 013** both touch security-facing description text; 013's ADR
  references 012's suffix registry. Either order.
- Everything else is independent.

- Operator action attached to 004: the Dockhand account name previously
  published in `.env.example` has been public — **rotate that account's
  password**. This is an ops task, not a repo change.

## Follow-ups awaiting a maintainer decision (audited, verified, not planned)

Security-lens items (from the security run):

- **Tool annotations** (`readOnlyHint` / `destructiveHint` / `idempotentHint`)
  derived from the already-generated `TOOL_ENDPOINT_MAP` — 42 destructive
  Docker operations are currently indistinguishable from `list_containers` to
  any client. Note the tension to resolve: `src/openapi/description-suffixes.ts`
  records a deliberate decision that *prose* destructiveness warnings are not
  worth it; structured annotations are a different channel and that rationale
  does not automatically carry over.
- **Read-only / area-scoped deployment profiles** — needs the generator
  problem solved first (skipping `registerTool` calls changes what the
  source-scanning generators see and breaks the CI derived-docs gate), which
  is why plan 013 gates at call time instead.
- **Credential-use-against-a-caller-chosen-host** — `test_git_repository_connection`
  and `update_git_repository` / `update_registry` let a stored deploy key or
  registry password be presented to an arbitrary host, bypassing Dockhand's
  "secrets are never readable back" posture. An optional host allowlist is the
  fix; one upstream behavior check is needed first (does Dockhand present the
  credential before or after validating the host?).
- **Per-caller attribution** — every MCP session acts as the same Dockhand
  admin, and Dockhand's audit log cannot attribute a change to an MCP session.
  Adding an allowlisted `target` field to the tool log lines would partially
  reverse a deliberate no-values logging decision; needs a call.
- **`create_auth_token` as an omission candidate** — it mints a long-lived
  plaintext token into the transcript that outlives `DOCKHAND_PASSWORD`
  rotation. `POST /api/auth/login` is already in the omission registry for a
  closely related reason. Plan 012 warns; omitting is the stronger option.
- **`create_volume` `driverOpts` and `update_environment` `socketPath`** —
  same host-escape family as plan 013, deliberately left out of its scope.
- **`get_runtime_stats` cross-session `lastError`** — real but already
  bounded and query-redacted, with the tradeoff documented at length in
  `src/utils/runtime-stats.ts`. Cheapest close: drop `message` from the tool
  response (it is already in the structured log with full `sid`/`call`
  context). Low priority.
- **Finite `MCP_MAX_SESSIONS` default** — plan 001 removes the leak that made
  this urgent; a cap is still worth deciding.

From the deep run:

- **Tool-group filtering for token cost** — 306 tools ≈ 38k tokens of schema
  in every conversation; 8.7 MB re-registered per session.
- **Runtime test coverage** — 401-relogin never actually exercised, SSE
  parsing at 0%, 205/306 tools never named in a test, 16 files assert on
  source text via regex, no coverage gate in CI.
- **SSE bounding** — cap deploy-stream output; return partial output on the
  5-minute timeout instead of discarding everything.
- **Agent-docs refresh** — `CLAUDE.md` / copilot-instructions / README tool
  reference are materially wrong (Zod v3 claim, 130+ vs 306 tools, Dependabot
  vs Renovate, README documents 205 of 308 tools, `fork-kit/README.md` does
  not exist, completed `docs/plans/` still read as open work).
- **Build-tooling robustness** — replace the regex tool-extractor with
  runtime-sourced data; precondition for table-driven registration and for
  registration-time tool filtering.
- **Renovate `customManagers` regex** for `Jenkinsfile` and
  `scripts/lint-in-container.sh` — no manager currently tracks those pins.
- **Scheduled image rebuild** for base-image CVEs (nothing rebuilds unless a
  `feat`/`fix` commit lands).
- **pino 9→10** — the open `renovate/pino-10.x` branch is a verified
  zero-source-change migration.
- **`logout` on a shared client** — destroys the upstream cookie for every
  session; decide whether it should invalidate locally or be removed.

## Findings considered and rejected (so nobody re-audits them)

- `exec_container` sending `{ envId }` instead of `{ env }` — matches the
  pinned upstream spec; deliberate and tested.
- Opt-in transport security defaults (no auth / Host check out of the box) and
  binding `0.0.0.0` — documented deliberate decisions in README "Securing the
  transport". The real gap (the shipped compose file made opting in
  impossible) is plan 004.
- Dev-tree `npm audit` advisories (7, all inside semantic-release's bundled
  npm) — not runtime-reachable. The *gate scoping* that lets them block the
  Jenkins sync is plan 008, step 4.
- Bearer-token length leak in `timingSafeTokenMatch` — real but negligible
  (leaks token length only, bounded by network jitter). Fold into a future
  transport-guard change rather than its own plan.
- Stored credentials readable back through a GET tool — **checked and clean**:
  git credentials, registries, Hawser/auth tokens, secret providers,
  notifications, LDAP/OIDC all strip or mask upstream. The exposure is
  credential *use* (follow-up) and *minting* (plan 012), not read-back.
- Backup destinations returning decrypted cloud credentials — no backup tools
  are registered today, so there is no exposure; worth an omission-registry
  note before that family is ever added.
- Path traversal into Dockhand — `encodePath()` covers every interpolated
  segment (337 call sites) and is guarded by `tests/path-encoding.test.ts`.
  Path *values* passed as query params are Dockhand's validation boundary;
  that concern is plan 013 instead.
- Prompt-injection content inside this repository — scanned across source,
  docs, README, CLAUDE.md, plans and config in both runs: none found.
- SDK dependency bloat (hono, jose, express in the prod tree) — not
  actionable from this repo; upstream SDK packaging.
- Downgrading TypeScript 7 to reunify the lint toolchain — the container split
  is the right call while typescript-eslint lacks TS7 support.
- Prometheus `/metrics` endpoint — `get_runtime_stats` plus the access log
  already cover it, and an unauthenticated endpoint would reopen the exposure
  surface the code deliberately closed.
- Chasing test flakiness — the suite is clean (env/global stubs restored,
  distinct e2e ports, ~1.3s full run).
