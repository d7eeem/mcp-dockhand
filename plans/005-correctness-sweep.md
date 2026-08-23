# Plan 005: Correctness sweep — six small verified bugs, each with its own test

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. Each step is independent — if one step hits a STOP condition,
> report it and continue with the remaining steps. When done, update the
> status row for this plan in `plans/README.md` — unless a reviewer dispatched
> you and told you they maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/tools/environments.ts src/tools/audit.ts src/tools/system.ts src/tools/meta.ts src/openapi/spec-loader.ts src/index.ts scripts/ tests/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch for a given step, treat it as that step's STOP condition.

## Status

- **Priority**: P2
- **Effort**: M (six independent S-sized fixes)
- **Risk**: LOW
- **Depends on**: none
- **Category**: bug
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

Six small, independently verified bugs. None is catastrophic; all produce
silently wrong behavior that costs debugging time downstream: a rename that
doesn't rename, an environment created with an empty host, pagination
parameters dropped, an updater that says "no update" wrongly, a corrupt spec
file that turns every session into a 500, and CI scripts whose entry-point
guard fails open under exotic checkout paths.

## Current state (per step)

Repo conventions for all steps: strict TS; tool validation errors are thrown
`Error`s (converted by `registerTool`'s shared catch); pure helpers get unit
tests; several tests regex-match tool source text, so keep line structure
stable where possible and run the full suite after each step.

**(a) `update_environment` — explicit `name` loses to `additionalSettings`.**
`src/tools/environments.ts:122-125`:

```typescript
      const body: Record<string, unknown> = {};
      if (name) body.name = name;
      // Merge additional settings first so explicit fields can override them
      if (additionalSettings) Object.assign(body, additionalSettings);
```

`body.name` is written *before* the merge, so `additionalSettings.name`
overrides the explicit argument — the comment states the opposite intent, and
every other explicit field (`icon` etc., lines 126-131) is correctly assigned
after the merge.

**(b) `resolveHostPort` — `host:port` yields an empty host.**
`src/tools/environments.ts:39-63`. `new URL('docker-host:2376')` does NOT
throw in Node: it parses as scheme `docker-host:` with `hostname === ''` and
`port === ''`, so the code sets `body.host = ''` and (with `useDefaultPort`)
`body.port = 2376`, never reaching the `tcp://` retry. Only digit-leading
forms like `192.168.1.5:2376` throw and get the correct fallback. Verified:

```
new URL('docker-host:2376') → {protocol:'docker-host:', hostname:'', port:'', pathname:'2376'}
new URL('192.168.1.5:2376') → throws
```

Reached from `create_environment`, `update_environment`,
`test_environment_connection`.

**(c) Falsy arguments dropped.**
`src/tools/audit.ts:19-20`:

```typescript
      if (limit) params.limit = limit;
      if (offset) params.offset = offset;
```

`offset: 0` / `limit: 0` are silently discarded (offset 0 is the natural
first page). And `src/tools/system.ts:292` (`reset_scanner_settings`):

```typescript
      if (removeImages) query.removeImages = 'true';
```

`removeImages: false` sends nothing — but the parameter is the tool's explicit
confirmation flag, so an explicit `false` should reach the server as `false`
(letting Dockhand refuse), not vanish. Note: `delete_container`'s
`if (force)` and `test_git_repository_connection`'s `if (credentialId)` were
reviewed and left alone — `false`/`0` are genuine no-ops there.

**(d) `compareSemver` — `NaN` on any non-numeric component.**
`src/tools/meta.ts:83-91`:

```typescript
export function compareSemver(a: string, b: string): -1 | 0 | 1 {
  const pa = a.replace(/^v/, '').split('.').map(Number);
  const pb = b.replace(/^v/, '').split('.').map(Number);
  for (let i = 0; i < 3; i++) {
    const d = (pa[i] ?? 0) - (pb[i] ?? 0);
    if (d !== 0) return d > 0 ? 1 : -1;
  }
  return 0;
}
```

`"1.2.3-beta.1"` → `[1,2,NaN]` → `d = NaN` → returns `-1` at that position
regardless of order. Consumer: `meta.ts:146`
`updateAvailable: compareSemver(latest, deps.current) > 0`, where `latest` is
a GitHub `tag_name` and `current` defaults to `'0.0.0-dev'`.

**(e) Corrupt spec file fails every session, unlike a missing one.**
`src/openapi/spec-loader.ts:62`:

```typescript
  cachedSpec = JSON.parse(readFileSync(SPEC_FILE, 'utf8')) as OpenApiSpec;
```

The missing-file path (lines 50-60) logs once and degrades to fallback
descriptions; a present-but-unparseable file throws out of `describeTool()` →
`registerAllTools()` → every `initialize` becomes a 500, retried forever.

**(f) CI scripts' main-guard fails open on encoded paths.**
Five scripts use `if (import.meta.url === \`file://${process.argv[1]}\`)`:
`scripts/validate-mcp-tools.mjs:1228`, `scripts/generate-tool-endpoint-map.mjs:244`,
`scripts/generate-coverage-doc.mjs:82`, `scripts/generate-body-contract-doc.mjs:105`,
`scripts/fetch-openapi.mjs:175`. `import.meta.url` percent-encodes characters
a raw path does not (spaces, `#`, non-ASCII); under such a checkout path the
guard is false, `main()` never runs, and the script **exits 0 having done
nothing** — a validation gate that fails open.

**(g) `MCP_PORT` — `parseInt` NaN binds a random port.**
`src/index.ts:55`:

```typescript
  port: parseInt(process.env['MCP_PORT'] ?? '8080', 10),
```

A typo (`8O80`) yields `NaN`; `app.listen(NaN, ...)` binds an ephemeral port
while the file otherwise fails loudly on bad config (`getEnvOrThrow`,
lines 45-54, with `logFatalSync` + `process.exit(1)`). The exemplar parser is
`parsePositiveInteger` in `src/session-lifecycle.ts:38-42`.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Typecheck tests | `npm run typecheck:tests` | exit 0 |
| One file | `npx vitest run tests/<file>` | all pass |
| Full tests | `npm test` | all pass |
| Derived docs | `npm run api:coverage-doc && npm run api:tool-endpoint-map && node scripts/generate-body-contract-doc.mjs && git diff --exit-code -- docs/coverage.md src/openapi/tool-endpoint-map.ts docs/body-contract-report.md` | no diff |

## Scope

**In scope**: `src/tools/environments.ts`, `src/tools/audit.ts`,
`src/tools/system.ts`, `src/tools/meta.ts` (compareSemver only),
`src/openapi/spec-loader.ts`, `src/index.ts`, the five `scripts/*.mjs`
main-guard lines, and new/extended tests:
`tests/environments-update-and-hostport.test.ts` (create),
`tests/falsy-params.test.ts` (create), `tests/tools/meta.compare-semver.test.ts`
(create or extend an existing meta test file), `tests/spec-loader.test.ts`
(extend), `tests/spec-loader-missing-spec.test.ts` (reference only),
`tests/js-scan.test.ts` (untouched).

**Out of scope**: everything else in `meta.ts` (a god-module split is a
separate refactor); `delete_container`/`credentialId` falsy sites (reviewed,
correct); `MCP_HOST` validation; any change to what `resolveHostPort` writes
for inputs that already work (`tcp://host:port`, bare `host` param).

## Git workflow

- Branch: `advisor/005-correctness-sweep`.
- One commit per step, conventional scopes:
  `fix(environments): …` (a, b), `fix(audit): …` / `fix(system): …` (c),
  `fix(tools): compareSemver handles pre-release tags` (d),
  `fix(api): degrade on corrupt openapi spec instead of failing sessions` (e),
  `fix(api): close the fail-open main guard in generator scripts` (f),
  `fix(system): validate MCP_PORT at startup` (g — scope `system` fits best).
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1 (a): reorder `update_environment` body assembly

Move `if (name) body.name = name;` to *after* the
`if (additionalSettings) Object.assign(body, additionalSettings);` line, so it
sits with the other explicit fields (before `icon`). Keep the comment — it is
now true. Test (in `tests/environments-update-and-hostport.test.ts`, harness
pattern: `tests/stack-env-merge-behavior.test.ts`): invoke the captured
`update_environment` handler with
`{ environmentId: 1, name: 'renamed', connectionType: 'hawser-edge', additionalSettings: { name: 'sneaky', foo: 1 } }`
against a mock client; assert `client.put` received a body with
`name: 'renamed'` and `foo: 1`.

**Verify**: `npx vitest run tests/environments-update-and-hostport.test.ts` → pass.

### Step 2 (b): fix `resolveHostPort` parsing

Replace the try/catch-cascade (lines 39-63) with: if `args.url` contains
`://`, parse it directly with `new URL(...)`; otherwise parse
`new URL('tcp://' + args.url)`. After either parse, if `parsed.hostname` is
empty, throw the existing error message. Keep the `port`/`useDefaultPort`
logic identical. Add cases to the same test file (call the captured
`create_environment` handler): `url: 'docker-host:2376'` →
`client.post` body has `host: 'docker-host'`, `port: 2376`;
`url: 'tcp://docker-host:2376'` → same; `url: '192.168.1.5:2376'` → host/port
correct; `url: ':::'` (or another unparseable form) → error response, no
`client.post` call. IPv6: `url: '[::1]:2376'` — Node's URL gives
`hostname === '[::1]'`; assert whatever the current behavior produces and pin
it with a comment (do not invent normalization).

**Verify**: same test file → all cases pass; `npm test` → green.

### Step 3 (c): stop dropping falsy params

`src/tools/audit.ts:19-20` → `if (limit !== undefined) ...`,
`if (offset !== undefined) ...`.
`src/tools/system.ts:292` → `if (removeImages !== undefined) query.removeImages = String(removeImages);`
Tests in `tests/falsy-params.test.ts` (same harness): `get_audit_log({ limit: 0, offset: 0 })`
→ `client.get` called with `{ limit: 0, offset: 0 }`;
`reset_scanner_settings({ environmentId: 1, removeImages: false })` →
`client.delete` query contains `removeImages: 'false'`.

**Verify**: `npx vitest run tests/falsy-params.test.ts` → pass.

### Step 4 (d): pre-release-safe `compareSemver`

Rewrite to strip from the first `-` or `+` for the numeric comparison, parse
with `Number.parseInt(x, 10)` defaulting to 0 on NaN, and when the numeric
triples are equal, rank "has pre-release" below "no pre-release" (that is
enough for the `updateAvailable` use; full pre-release ordering is out of
scope — say so in a comment). Tests (new file
`tests/tools/meta.compare-semver.test.ts`, matching the naming style of the
existing `tests/tools/meta.*.test.ts` files): `('1.2.3','1.2.3') → 0`;
`('v1.3.0','1.2.9') → 1`; `('1.2.3-beta.1','1.2.3') → -1`;
`('1.2.4-beta.1','1.2.3') → 1`; `('1.2.3','0.0.0-dev') → 1`.
`compareSemver` is already exported.

**Verify**: `npx vitest run tests/tools/meta.compare-semver.test.ts` → pass.

### Step 5 (e): guard the spec parse

Wrap line 62 of `src/openapi/spec-loader.ts` in try/catch: on failure, log a
once-per-process `logger.warn` (mirror the `loggedMissing` pattern at lines
50-57, with its own boolean, message "spec file is not valid JSON — tool
descriptions will use the fallback text"), set `cachedSpec = null`, return it.
Extend `tests/spec-loader.test.ts` (see how it and
`tests/spec-loader-missing-spec.test.ts` fixture the spec path): a spec file
containing `{invalid` → `loadSpec()` returns null, warns once across two
calls, and does not throw.

**Verify**: `npx vitest run tests/spec-loader.test.ts` → pass.

### Step 6 (f): fail-closed main guards

In all five scripts, add `import { pathToFileURL } from 'node:url';` and
change the guard to
`if (import.meta.url === pathToFileURL(process.argv[1]).href) {`.
No behavior change on normal paths. The generator outputs must be
byte-identical: run the derived-docs command → no diff.

**Verify**: derived-docs command → exit 0, no diff; `npm test` → green (the
`tests/validate-mcp-tools.test.ts` suite imports these modules and must not
see side effects — the guard still prevents `main()` on import).

### Step 7 (g): validate `MCP_PORT`

In `src/index.ts`, replace line 55's bare `parseInt` with a checked parse:
parse with `Number.parseInt`, and if not an integer in `[1, 65535]`, call the
existing `logFatalSync({ component: 'config', variable: 'MCP_PORT' }, 'MCP_PORT must be an integer between 1 and 65535')`
and `process.exit(1)` — mirroring `getEnvOrThrow` directly above. Keep the
`8080` default for unset. Testing startup exits is already modeled in
`tests/index-fatal-exit-logging.test.ts` — extend it if the harness reaches
this code path cheaply; if it mocks too early to reach the port parse, add
the validation as a small exported helper (e.g. `parsePort(value: string | undefined): number`)
in `src/index.ts` is NOT importable without side effects — put the helper in
`src/session-lifecycle.ts`? No — out of scope. Instead create
`src/utils/parse-port.ts` (new small util, exported pure function) and test it
directly in `tests/utils/parse-port.test.ts`: `undefined → 8080`,
`'9090' → 9090`, `'8O80' → throws/sentinel`, `'0' → invalid`, `'70000' → invalid`.
Wire `index.ts` to call it and exit on the error.

**Verify**: `npx vitest run tests/utils/parse-port.test.ts` → pass;
`npm run typecheck` → exit 0.

## Test plan

Summarized per step above: 4 new test files + 2 extended, each patterned on a
named existing test. Final gate: `npm test` all green, derived-docs no diff.

## Done criteria

- [ ] All seven step verifications pass
- [ ] `npm run typecheck` + `npm run typecheck:tests` exit 0; `npm test` exits 0
- [ ] `grep -n "if (name) body.name" src/tools/environments.ts` shows it AFTER the additionalSettings merge
- [ ] `grep -c "pathToFileURL(process.argv\[1\])" scripts/*.mjs` → 5
- [ ] Derived-docs regeneration produces no diff
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- Any excerpt mismatch on the step you are executing (drift) — skip that step,
  report, continue with the others.
- Step 2: an existing test pins the current empty-host behavior (would mean
  something depends on it) — report before changing.
- Step 3: `docs/dockhand-openapi.json` shows `removeImages` as a boolean query
  param that Dockhand rejects when `'false'` — if the spec says
  presence-only semantics, keep `if (removeImages)` and instead make the Zod
  schema `z.literal(true)` so `false` is rejected client-side; report which
  branch you took.
- Step 6: any generator output changes byte-wise — the guard edit must be
  behavior-neutral; report the diff.

## Maintenance notes

- Step 2's IPv6 pin is deliberate documentation of current behavior, not an
  endorsement — revisit if Dockhand grows IPv6 host support.
- Step 4 deliberately implements only "pre-release < release", not full
  SemVer 2 precedence; if the updater ever compares two pre-releases, replace
  with a real semver library decision.
- The `parse-port` util is the seed of the central-config module recorded as
  a follow-up in plans/README.md (env reads are currently scattered across
  seven files); fold it in when that lands.
