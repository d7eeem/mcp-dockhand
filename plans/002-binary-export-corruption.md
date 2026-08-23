# Plan 002: Make export_image and export_volume return usable archives, and bound raw downloads

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/tools/images.ts src/tools/volumes.ts src/tools/containers.ts src/client/dockhand-client.ts tests/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: bug
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

`export_image` and `export_volume` fetch tar/gzip archives through the JSON/text
request path: the client sees a non-JSON content type and does
`await response.text()`, which lossily UTF-8-decodes the binary bytes. Both
tools therefore return **silently corrupted archives** — a 2xx result whose
payload cannot be untarred, with no error and no warning. The repo already has
the correct pattern in `download_container_file` (raw buffer → base64).
Separately, all three raw-download paths are unbounded: a large file is
base64-inflated (+33%) straight into the model's context with no size guard.
This plan fixes the corruption and adds one shared size cap.

## Current state

Relevant files:

- `src/tools/images.ts:90-98` — `export_image` uses `client.get(...)`:

```typescript
  registerTool(server, 'export_image',
    {
      environmentId: z.number().describe('Environment ID'),
      imageId: z.string().describe('Image ID'),
    },
    async ({ environmentId, imageId }) => {
      return jsonResponse(await client.get(`/api/images/${encodePath(imageId)}/export`, { env: environmentId }));
    }
  );
```

- `src/tools/volumes.ts:90-99` — `export_volume`, identical shape:

```typescript
  registerTool(server, 'export_volume',
    {
      environmentId: z.number().describe('Environment ID'),
      volumeName: z.string().describe('Volume name'),
    },
    async ({ environmentId, volumeName }) => {
      return jsonResponse(await client.get(`/api/volumes/${encodePath(volumeName)}/export`, { env: environmentId }));
    }
  );
```

- `src/client/dockhand-client.ts:513-520` — the text fallback these hit today
  (inside `request()`): non-JSON content type → `await response.text()`.

- `src/client/dockhand-client.ts:91-95` — `getRaw()`, the correct primitive:

```typescript
  async getRaw(path: string, params?: Record<string, string | number | undefined>): Promise<Buffer> {
    const url = this.buildUrl(path, params);
    const response = await this.requestRaw('GET', url);
    return Buffer.from(await response.arrayBuffer());
  }
```

- `src/tools/containers.ts:380-392` — `download_container_file`, the exemplar
  return shape to copy:

```typescript
    async ({ environmentId, containerId, path }) => {
      const buffer = await client.getRaw(`/api/containers/${encodePath(containerId)}/files/download`, {
        env: environmentId,
        path,
      });
      return textResponse(`base64:${buffer.toString('base64')}`);
    }
```

- The pinned spec (`docs/dockhand-openapi.json`) documents both export
  endpoints' 200 responses as binary (`application/x-tar` /
  `application/gzip` / `application/octet-stream`).

Conventions:

- Tool responses go through `jsonResponse`/`textResponse`/`errorResponse` from
  `src/utils/tool-helper.js` re-exports (all tool files import from
  `./…/utils/tool-helper.js`) — never inline `content: [...]` literals.
- `encodePath()` on every interpolated path segment.
- **Generated-artifact gate**: tool descriptions and the endpoint map are
  derived by regex-scanning tool source (`scripts/validate-mcp-tools.mjs`).
  The `client.getRaw(` call with an inline template-literal path keeps the
  extractor working (same shape as `download_container_file`). Do NOT hoist
  the path into a variable and do NOT add a generic type argument — either
  makes the tool invisible to the extractor and fails the CI derived-docs
  gate.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Typecheck tests | `npm run typecheck:tests` | exit 0 |
| Full tests | `npm test` | all pass |
| Derived docs still in sync | `npm run api:coverage-doc && npm run api:tool-endpoint-map && node scripts/generate-body-contract-doc.mjs && git diff --exit-code -- docs/coverage.md src/openapi/tool-endpoint-map.ts docs/body-contract-report.md` | exit 0, no diff |

## Scope

**In scope** (the only files you should modify):
- `src/tools/images.ts` (export_image only)
- `src/tools/volumes.ts` (export_volume only)
- `src/tools/containers.ts` (download_container_file — size cap only)
- `tests/binary-export-tools.test.ts` (create)

**Out of scope** (do NOT touch):
- `src/client/dockhand-client.ts` — `getRaw()` is correct as is; the cap lives
  in the tools so the error message can name the tool-level remedy.
- `import_image` / upload paths — different direction, not broken.
- `src/openapi/tool-endpoint-map.ts`, `docs/coverage.md` — generated; only
  regenerate via the commands above if the gate asks for it (it should not:
  method and path are unchanged).

## Git workflow

- Branch: `advisor/002-binary-exports`.
- Commits (scopes must come from `.commitlintrc.json`):
  - `fix(images): return export_image archive as base64 instead of mangled text`
  - `fix(volumes): return export_volume archive as base64 instead of mangled text`
  - `fix(containers): cap raw download size before base64-inflating into context`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Add a shared size-cap constant

In `src/tools/containers.ts`, near the top with the other module constants,
add:

```typescript
// Raw downloads are base64-inflated (+33%) into the MCP client's context.
// Beyond this many raw bytes the result is refused with an explanatory error
// instead of flooding the context (10 MiB raw ≈ 13.9 MiB base64).
export const MAX_RAW_DOWNLOAD_BYTES = 10 * 1024 * 1024;
```

Export it so `images.ts` and `volumes.ts` can import it (import from
`'./containers.js'`).

**Verify**: `npm run typecheck` → exit 0.

### Step 2: Fix `export_image`

Replace the handler body in `src/tools/images.ts:90-98` (keep the schema):

```typescript
    async ({ environmentId, imageId }) => {
      const buffer = await client.getRaw(`/api/images/${encodePath(imageId)}/export`, { env: environmentId });
      if (buffer.length > MAX_RAW_DOWNLOAD_BYTES) {
        return errorResponse(
          `Image export is ${buffer.length} bytes, larger than the ${MAX_RAW_DOWNLOAD_BYTES}-byte limit for inline results. Export it on the Dockhand host instead.`,
        );
      }
      return textResponse(`base64:${buffer.toString('base64')}`);
    }
```

Add the needed imports (`errorResponse`, `textResponse` are exported from the
same helper module the file already imports from; add
`MAX_RAW_DOWNLOAD_BYTES` from `'./containers.js'`). Remove `jsonResponse` from
the import only if it is now unused in the file (it will not be — other tools
use it).

**Verify**: `npm run typecheck` → exit 0.

### Step 3: Fix `export_volume`

Same transformation in `src/tools/volumes.ts:90-99`, with
`/api/volumes/${encodePath(volumeName)}/export`.

**Verify**: `npm run typecheck` → exit 0.

### Step 4: Cap `download_container_file`

In `src/tools/containers.ts:380-392`, after the `getRaw` call, add the same
`buffer.length > MAX_RAW_DOWNLOAD_BYTES` guard returning `errorResponse` with
the actual size, the limit, and the path that was requested (no file
*content* in the error).

**Verify**: `npm run typecheck` → exit 0.

### Step 5: Tests

Create `tests/binary-export-tools.test.ts`. Pattern: the fake-server +
handler-capture approach of `tests/stack-env-merge-behavior.test.ts` (register
the tool module against a stub `{ tool: (name, desc, schema, cb) => ... }`
server, capture the callbacks by name, invoke them with a mock client).

Mock client: an object whose `getRaw` is a `vi.fn()` returning a `Buffer`
containing bytes that are NOT valid UTF-8 round-trippable (e.g.
`Buffer.from([0x1f, 0x8b, 0x08, 0x00, 0xff, 0xfe])` — a gzip magic prefix plus
invalid continuation bytes).

Cases:

1. `export_image` calls `client.getRaw` (not `client.get`) with
   `/api/images/<id>/export` and `{ env }`; the returned text starts with
   `base64:` and `Buffer.from(text.slice(7), 'base64')` byte-equals the mock
   buffer (round-trip proof — this is the assertion that fails on the old
   text path).
2. Same for `export_volume`.
3. `export_image` with a buffer of `MAX_RAW_DOWNLOAD_BYTES + 1` bytes returns
   `isError: true` and a message naming both numbers.
4. `download_container_file` over the cap returns `isError: true`.

**Verify**: `npx vitest run tests/binary-export-tools.test.ts` → all pass;
then `npm test` → all pass (watch `tests/api-contracts.test.ts` and
`tests/tool-endpoint.test.ts` — they cross-check tool↔endpoint mappings and
must stay green since method/path are unchanged).

### Step 6: Derived-docs gate

Run the derived-docs command from the table. Expected: no diff (the extractor
sees `client.getRaw(` with an inline path, same endpoint as before). If a diff
appears in `src/openapi/tool-endpoint-map.ts` for these two tools, inspect it:
the entry must still map `export_image → GET /api/images/{id}/export` and
`export_volume → GET /api/volumes/{name}/export`. Any other change → STOP.

**Verify**: `git diff --exit-code -- docs/coverage.md src/openapi/tool-endpoint-map.ts docs/body-contract-report.md` → exit 0.

## Test plan

Covered in steps 5–6. New tests: 4 cases in `tests/binary-export-tools.test.ts`,
modeled on `tests/stack-env-merge-behavior.test.ts`.

## Done criteria

- [ ] `npm run typecheck` and `npm run typecheck:tests` exit 0
- [ ] `npm test` exits 0 including the 4 new tests
- [ ] `grep -n "client.get(" src/tools/images.ts src/tools/volumes.ts | grep -c export` returns 0 (no export tool uses the text path)
- [ ] Derived-docs regeneration produces no diff
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- The excerpts no longer match (drift).
- The derived-docs gate shows changes beyond the two export tools' entries.
- `getRaw()`'s signature has changed in `src/client/dockhand-client.ts`
  (currently `(path, params?) => Promise<Buffer>`).
- You find the Dockhand export endpoints actually return JSON envelopes in the
  current spec (check `docs/dockhand-openapi.json` for
  `/api/images/{id}/export` and `/api/volumes/{name}/export` response content
  types) — if they are JSON now, the premise is wrong; report instead of
  changing anything.

## Maintenance notes

- If a future MCP SDK version supports binary/resource content blocks,
  replace the `base64:` text convention with a proper resource response for
  all three tools at once.
- The cap constant is deliberately in `containers.ts` next to its first
  consumer rather than a new config surface; if an operator-facing knob is
  wanted later, route it through the (planned) central config module rather
  than a fourth ad-hoc `process.env` read.
- Reviewer: check that the error responses contain sizes and paths only —
  never file content.
