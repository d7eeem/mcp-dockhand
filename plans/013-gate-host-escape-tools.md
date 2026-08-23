# Plan 013: Put the host-escape capabilities behind an explicit opt-in

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/tools/containers.ts src/tools/system.ts src/tools/index.ts tests/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P2
- **Effort**: M
- **Risk**: MED — this plan changes defaults. Read "Compatibility stance"
  before writing any code; getting that wrong breaks working deployments.
- **Depends on**: none
- **Category**: security
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

Two capabilities in this server are not "manage Docker" — they are "control
the host", and neither is distinguishable from an ordinary tool call at the
approval prompt:

1. **`update_container` / `create_container` accept container settings that
   escape the container boundary.** The `settings` allowlist permits
   `privileged`, `capAdd`, `devices`, `securityOpt`, `pidMode`, `ipcMode`,
   `usernsMode`, `cgroupParent`, `sysctls`, `runtime`, and `volumeBinds`. The
   allowlist's own comment makes clear it is a **typo guard**, not a security
   boundary: it exists because Dockhand silently drops keys that don't match
   its `CreateContainerOptions` shape. An approved "container update" can
   therefore mount the host filesystem or run privileged.

2. **`list_system_files` / `get_system_file_content` / `write_system_file`
   read and write the Dockhand *server's own* filesystem** — not a container's
   — with the path forwarded verbatim. That is a general-purpose file reader on
   the machine that also holds Dockhand's database and its encrypted
   credential store, and it has no relationship to Docker management.

This repo already has a precedent and a mechanism for exactly this judgment:
`docs/adr/0001-omission-registry.md` plus `docs/omitted-endpoints.json` record
deliberately un-mirrored endpoints, and `POST /api/self-update` is omitted on
this reasoning. Arbitrary host-file access never went through that process.

## Compatibility stance (read before coding)

These tools work today and someone may depend on them. This plan therefore
**does not remove or block anything by default**. It:

- adds two opt-in environment flags that *enable* the risky surface,
- defaults both to **permissive**, preserving today's behavior exactly,
- makes the permissive path **log a one-time warning** naming the flag,
- and documents the intent to flip the defaults in the next major.

If you find yourself writing code that rejects a call under default
configuration, you have misread this section — stop and re-read it.

## Current state

`src/tools/containers.ts:28-44` — the allowlist (abridged; read the full
comment at lines 12-27, which states it is a typo guard aligned to upstream
`CreateContainerOptions`):

```typescript
const UPDATE_CONTAINER_ALLOWED_SETTINGS_KEYS = new Set([
  // top-level control flags, not part of CreateContainerOptions
  'startAfterUpdate', 'repullImage',
  // CreateContainerOptions fields
  'name', 'image', 'ports', 'volumes', 'volumeBinds', 'env', 'labels', 'cmd',
  'entrypoint', 'workingDir', 'restartPolicy', 'restartMaxRetries', 'networkMode',
  ... 'user', 'privileged',
  ... 'capAdd', 'capDrop', 'devices', 'dns', 'dnsSearch',
  'dnsOptions', 'securityOpt', 'ulimits', ... 'sysctls', 'logDriver', 'logOptions', 'ipcMode',
  'pidMode', 'utsMode', 'hostname', 'cgroupParent', 'stopSignal', 'init',
  ... 'runtime',
  'readonlyRootfs', 'cpusetCpus', 'cpusetMems', 'groupAdd', 'memorySwappiness',
  'usernsMode', 'domainname',
]);
```

The enforcement point is in `update_container`'s handler around
`src/tools/containers.ts:222-237` — read it; it rejects unknown keys with an
error naming them. That rejection path is the model for what you will add.

`src/tools/system.ts:51-70` and `:275-282` — the host-file tools:

```typescript
  registerTool(server, 'list_system_files',
    {
      path: z.string().optional().describe('Directory path'),
    },
    async ({ path }) => {
      return jsonResponse(await client.get('/api/system/files', path ? { path } : undefined));
    }
  );

  registerTool(server, 'get_system_file_content',
    {
      path: z.string().describe('File path'),
    },
    async ({ path }) => {
      return textResponse(await client.get('/api/system/files/content', { path }));
    }
  );
```

```typescript
  registerTool(server, 'write_system_file',
    {
      path: z.string().describe('Absolute path on the Dockhand server to create as a directory'),
    },
    async ({ path }) => {
      return jsonResponse(await client.post('/api/system/files', { path }));
    }
  );
```

`src/tools/index.ts:31-56` — `registerAllTools()` calls the 22 family
registrars unconditionally; there is no filtering mechanism of any kind today.

Conventions:

- Env parsing helpers with the right shape (empty string treated as unset)
  live in `src/session-lifecycle.ts:38-48` — read them as the pattern; do not
  add a fourth ad-hoc `process.env` style.
- Startup warnings: `src/server.ts:65-73` logs a `[security]` warning when the
  transport is unprotected — match that voice and level for the new warning.
- Description suffixes (`src/openapi/description-suffixes.ts`) are how
  caller-side operating rules get surfaced; plan 012 extends that registry, so
  coordinate if both are in flight.
- The build tooling regex-scans tool source for `registerTool(server, '<name>'`
  and `client.<verb>(`; conditional *registration* changes what the extractor
  sees. See the STOP condition about the derived-docs gate.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Typecheck tests | `npm run typecheck:tests` | exit 0 |
| Full tests | `npm test` | all pass |
| Derived docs | `npm run api:coverage-doc && npm run api:tool-endpoint-map && node scripts/generate-body-contract-doc.mjs && git diff --exit-code -- docs/coverage.md src/openapi/tool-endpoint-map.ts docs/body-contract-report.md` | no diff |

## Scope

**In scope**:
- `src/tools/containers.ts` — split the allowlist into tiers; warn (do not
  block) by default.
- `src/tools/system.ts` — warn on the host-file tools; honor the flag.
- `src/utils/env-helpers.ts` **or** a small new `src/utils/feature-flags.ts` —
  the two flag parsers (pick one home and say which).
- `README.md` — document both flags in the existing configuration table.
- `docs/adr/0003-host-escape-tool-gating.md` (create) — record the decision,
  following the format of `docs/adr/0002-description-override-map.md`.
- `tests/host-escape-gating.test.ts` (create).

**Out of scope**:
- **Removing** any tool, or blocking anything by default (see "Compatibility
  stance").
- Conditional *registration* (skipping `registerTool` calls) — that changes
  what the derived-docs generators see and would fail the CI gate. Gate at
  **call time**, not registration time. General tool-filtering is a separate
  recorded follow-up that must solve the generator problem first.
- `create_volume`'s `driverOpts` and `update_environment`'s `socketPath` —
  same family of concern, recorded as follow-ups; keep this plan's surface to
  the two verified items above.
- Path normalization/validation of the host paths — Dockhand's own validation
  boundary; do not reimplement it here.

## Git workflow

- Branch: `advisor/013-host-escape-gating`.
- Commits (scopes from `.commitlintrc.json`):
  - `feat(security): flag-gate privileged container settings`
  - `feat(security): flag-gate the Dockhand host-file tools`
  - `docs(security): record ADR-0003 for host-escape tool gating`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Add the two flags

Add parsers for:

- `MCP_ALLOW_PRIVILEGED_CONTAINERS` — default **true** (permissive today).
- `MCP_ALLOW_HOST_FILES` — default **true** (permissive today).

Use a boolean parser in the style of `src/session-lifecycle.ts:38-48`: empty
or unset → default; `'false'`/`'0'`/`'no'` → false; `'true'`/`'1'`/`'yes'` →
true; anything else → default plus a warning. Put both in one place and export
a small typed accessor, e.g. `getFeatureFlags(env = process.env)`.

**Verify**: `npm run typecheck` → exit 0.

### Step 2: Tier the container settings allowlist

In `src/tools/containers.ts`, add a second set naming the host-escape subset —
derive it from the existing allowlist, do not retype the whole thing:

```typescript
/**
 * The subset of UPDATE_CONTAINER_ALLOWED_SETTINGS_KEYS that crosses the
 * container boundary: each of these can give the container root-equivalent
 * access to the Docker host. The parent set is a TYPO guard aligned to
 * upstream's CreateContainerOptions (see its comment); this set is a
 * SECURITY tier, and the two are maintained for different reasons.
 */
const HOST_ESCAPE_SETTINGS_KEYS = new Set([
  'privileged', 'capAdd', 'devices', 'securityOpt', 'pidMode', 'ipcMode',
  'usernsMode', 'cgroupParent', 'sysctls', 'runtime', 'volumeBinds',
]);
```

Add a runtime check next to the existing unknown-key rejection in
`update_container`, and the equivalent point in `create_container` (find where
`volumes` → `HostConfig.Binds` and `networkMode` are assembled):

- When the flag is **true** (default): allow, and log **once per process** a
  `[security]` warning naming the keys used and the flag that will govern them
  in a future major. Use a module-scope `let warned = false` guard, mirroring
  the `loggedMissing` pattern in `src/openapi/spec-loader.ts`.
- When the flag is **false**: reject with an error naming the specific keys and
  the flag to set — matching the tone of the existing unknown-key rejection.

Do not change the parent allowlist's contents.

**Verify**: `npm run typecheck` → exit 0; `npx vitest run tests/update-container-contract.test.ts tests/update-container-runtime-fields.test.ts` → all pass (they encode the current contract; if one fails, you changed default behavior — fix that, do not edit the test).

### Step 3: Gate the host-file tools at call time

In `src/tools/system.ts`, in the three handlers (`list_system_files`,
`get_system_file_content`, `write_system_file`):

- Flag **true** (default): proceed, with the same once-per-process
  `[security]` warning naming `MCP_ALLOW_HOST_FILES`.
- Flag **false**: return an error explaining the tool is disabled and naming
  the flag. Throw `new Error(...)` — `registerTool`'s shared catch converts it
  to a proper MCP error response (see `src/utils/tool-helper.ts`).

Keep the `registerTool(...)` calls and the inline `client.<verb>(...)` calls
exactly as they are so the source extractor still sees them.

**Verify**: `npm run typecheck` → exit 0; derived-docs command → **no diff**.

### Step 4: Document

- Add both flags to the `README.md` configuration table (the table around
  lines 64-78), matching the existing column style and stating the current
  default is permissive.
- Write `docs/adr/0003-host-escape-tool-gating.md` following the structure of
  `docs/adr/0002-description-override-map.md`: Status/Date/Refs, Context (what
  the capability is and why it is not "manage Docker"), Decision (flags,
  permissive default now, intent to flip in the next major), Consequences,
  Links (the ADR-0001 omission-registry precedent, the files touched, the
  tests). Note: ADR 0001 is written in German and 0002 in English — match
  0002 (English), and add the new entry to `docs/adr/README.md`'s index.

**Verify**: `test -f docs/adr/0003-host-escape-tool-gating.md`;
`grep -n "0003" docs/adr/README.md` → present;
`grep -n "MCP_ALLOW_HOST_FILES" README.md` → present.

### Step 5: Tests

Create `tests/host-escape-gating.test.ts`, using the fake-server handler
harness (pattern: `tests/stack-env-merge-behavior.test.ts`) and the env-stubbing
approach used in `tests/session-lifecycle.test.ts`.

Cases:

1. **Default is permissive (container settings).** With no flag set,
   `update_container` with `settings: { privileged: true }` still forwards
   `privileged` to the client — today's behavior, unchanged.
2. **Flag off rejects.** With `MCP_ALLOW_PRIVILEGED_CONTAINERS=false`, the same
   call returns an error naming `privileged` and the flag, and the mock client
   was **not** called.
3. **Non-escape settings unaffected.** With the flag off,
   `settings: { memory: 512 }` still succeeds.
4. **Default is permissive (host files).** With no flag set,
   `get_system_file_content` calls the client as before.
5. **Flag off disables host-file tools.** With `MCP_ALLOW_HOST_FILES=false`,
   all three tools return an error naming the flag and the client is not
   called.
6. **The warning fires once**, not per call (capture log output with the
   `fs.writeSync` spy helper from `tests/session-lifecycle.test.ts:16-27`).

**Verify**: `npx vitest run tests/host-escape-gating.test.ts` → all pass;
`npm test` → all pass.

## Test plan

Six cases as above in `tests/host-escape-gating.test.ts`. The critical ones
are 1 and 4 — they pin that this plan did **not** change default behavior.

## Done criteria

- [ ] `npm run typecheck` and `npm run typecheck:tests` exit 0
- [ ] `npm test` exits 0 including 6 new tests
- [ ] Derived-docs regeneration produces **no diff**
- [ ] With no env flags set, no existing test changed its expectations
      (`git diff` on `tests/` shows only the new file, plus at most mechanical
      changes you can justify in the report)
- [ ] `grep -n "HOST_ESCAPE_SETTINGS_KEYS" src/tools/containers.ts` → present
- [ ] README table and `docs/adr/0003-*.md` + ADR index updated
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- The derived-docs gate produces a diff — you changed what the extractor sees;
  revert and gate at call time only.
- An existing container-contract test fails under **default** configuration —
  that means the default stopped being permissive. Fix the code, never the
  test.
- You conclude the flags should default to *restrictive* — that is a real
  argument, but it is a maintainer decision, not an executor one. Implement
  permissive-by-default as specified and record the argument in the ADR's
  Consequences section.
- The `create_container` assembly point does not exist in the shape described
  (drift) — gate `update_container` only, and report.

## Maintenance notes

- The two sets in `containers.ts` are maintained for different reasons and
  must not be merged: the parent allowlist tracks upstream's
  `CreateContainerOptions` (a typo guard, re-validated per Dockhand release),
  the new set is a security tier. Say so in the code comment, because the next
  person will be tempted to unify them.
- Flipping the defaults is a **major**-release change and needs a release note;
  the ADR should say so explicitly so the decision is reconstructable.
- Related follow-ups recorded in `plans/README.md`: `create_volume`'s
  `driverOpts`, `update_environment`'s `socketPath`, tool annotations, and a
  read-only deployment profile (which would subsume much of this at the
  registration layer once the generator problem is solved).
- Reviewer: verify by reading that no default-configuration path can now
  reject a call that previously succeeded.
