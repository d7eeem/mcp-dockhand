# Plan 006: Stop pretty-printing tool responses — compact JSON halves the tokens every result costs

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/utils/response.ts tests/`
> If `src/utils/response.ts` changed since this plan was written, compare the
> "Current state" excerpt against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P2
- **Effort**: S (the code change is 3 characters ×3; the work is the test fallout)
- **Risk**: LOW
- **Depends on**: none
- **Category**: perf (token cost)
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

Every one of the 306 tools returns through three helpers that
`JSON.stringify(data, null, 2)`. Two-space indentation nearly doubles the
byte size of deeply nested Docker payloads (measured on a comparable nested
JSON document: 23,480 bytes compact vs 46,542 pretty — +98%). The consumer is
an LLM, which reads compact JSON just as well; the extra bytes are pure token
cost paid on **every tool result in every conversation**. Nothing in the MCP
contract requires pretty-printing.

## Current state

`src/utils/response.ts` (entire file, 15 lines):

```typescript
/**
 * Standardized MCP response helpers to reduce duplication across tool files.
 */

export function jsonResponse(data: unknown) {
  return { content: [{ type: 'text' as const, text: JSON.stringify(data, null, 2) }] };
}

export function textResponse(data: unknown) {
  return { content: [{ type: 'text' as const, text: typeof data === 'string' ? data : JSON.stringify(data, null, 2) }] };
}

export function errorResponse(message: string) {
  return { content: [{ type: 'text' as const, text: JSON.stringify({ error: message }, null, 2) }], isError: true as const };
}
```

All 306 tools return through these (there are zero inline `content: [...]`
literals in `src/tools/` — verified during the audit). The risk surface is
tests that assert on the *formatted* string: expect failures in any test that
matches indented JSON output of a tool handler (candidates:
`tests/stack-env-merge-behavior.test.ts`, `tests/secret-providers.test.ts`,
`tests/update-container-runtime-fields.test.ts`, meta tool tests under
`tests/tools/` — the exact set is discovered by running the suite).

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Full tests | `npm test` | all pass (after test updates) |
| Derived docs | `npm run api:coverage-doc && npm run api:tool-endpoint-map && node scripts/generate-body-contract-doc.mjs && git diff --exit-code -- docs/coverage.md src/openapi/tool-endpoint-map.ts docs/body-contract-report.md` | no diff |

## Scope

**In scope**:
- `src/utils/response.ts`
- Test files whose assertions break **only because of formatting** (update the
  assertion to the compact form, or better: `JSON.parse` the text and assert
  on the object — that makes the test formatting-agnostic).

**Out of scope**:
- Any change to response *content* or shape — `content[0].type`, `isError`,
  key order, what is included.
- Adding an env var to re-enable pretty printing — not worth a config surface;
  a developer can pipe through `jq`.
- `src/utils/tool-helper.ts` and every file under `src/tools/` — nothing there
  should need to change; if it does, STOP.

## Git workflow

- Branch: `advisor/006-compact-json`.
- Commits: `perf(tools): return compact JSON from tool response helpers` and,
  if split, `test(tests): make response assertions formatting-agnostic`.
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Compact the three helpers

In `src/utils/response.ts`, change all three `JSON.stringify(data, null, 2)` /
`JSON.stringify({ error: message }, null, 2)` calls to single-argument
`JSON.stringify(...)`. Update the file's doc comment to note the choice:
"Compact JSON deliberately — the consumer is a model; indentation ~doubles
the token cost of nested Docker payloads."

**Verify**: `npm run typecheck` → exit 0.

### Step 2: Repair formatting-coupled tests

Run `npm test`. For each failure, confirm the assertion difference is
whitespace-only (the expected and received strings must parse to deep-equal
objects — check with `JSON.parse` mentally or in a scratch script). Preferred
repair: parse the handler's `result.content[0].text` and assert on the
object. Acceptable repair: update the expected literal to compact form.
Forbidden: loosening what the assertion checks.

**Verify**: `npm test` → all pass.

### Step 3: Derived docs unaffected

The generators scan tool source, not responses, so nothing should change.

**Verify**: derived-docs command → exit 0, no diff.

## Test plan

No new tests required; the change is covered by the entire existing
handler-test surface. If step 2 converts assertions to parse-then-compare,
note in the commit body that those tests are now formatting-agnostic.

## Done criteria

- [ ] `grep -c "null, 2" src/utils/response.ts` → 0
- [ ] `npm run typecheck` + `npm run typecheck:tests` exit 0; `npm test` exits 0
- [ ] Derived-docs regeneration produces no diff
- [ ] Only `src/utils/response.ts` and test files modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- A failing test's expected-vs-received difference is NOT whitespace-only —
  that would mean something besides formatting depends on the helpers; report
  it.
- Any file under `src/tools/` or `src/utils/tool-helper.ts` appears to need a
  change.
- More than ~15 test files break — the coupling is broader than the audit
  measured; report the list before mass-editing.

## Maintenance notes

- Future test authors: assert on parsed objects, not on `text` string
  literals — this change is the precedent.
- If a human-readable mode is ever genuinely wanted (debug sessions), add it
  in the client consuming the output, not in the server's response path.
