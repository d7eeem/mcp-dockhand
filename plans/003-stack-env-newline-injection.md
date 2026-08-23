# Plan 003: Reject newlines in stack .env keys and values before they are written

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/utils/env-helpers.ts src/tools/stacks.ts tests/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S
- **Risk**: LOW
- **Depends on**: none
- **Category**: security (input-contract hardening)
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

`update_stack_env` accepts arbitrary strings for env-var keys and values and
interpolates them verbatim into `.env` file lines that Dockhand then feeds to
docker compose. A value containing a line separator therefore writes
*additional* `KEY=VALUE` lines the caller never declared — a config-injection
path whose realistic trigger is an AI agent relaying a value it did not author
(from a file, an issue, an upstream API). The tool's own bookkeeping re-parses
the injected lines as real keys, so the reported added/updated/preserved
summary no longer matches what was written. Newlines have no legitimate
meaning in this format; rejecting them turns silent corruption into a loud
tool error.

## Current state

- `src/utils/env-helpers.ts:140` — updated lines are built by direct
  interpolation (inside `upsertDotEnv`):

```typescript
    seen.add(key);
    return `${prefix}${key}=${byKey.get(key)}`;
  });

  const appended = vars.filter((v) => !seen.has(v.key)).map((v) => `${v.key}=${v.value}`);
```

(line 143 is the `appended` map — same unescaped interpolation.)

- `src/tools/stacks.ts:165-171` — the schema accepts any string:

```typescript
      variables: z.array(z.object({
        key: z.string().describe('Environment variable name (UPPER_SNAKE_CASE convention)'),
        value: z.string().describe('Variable value as string'),
        isSecret: z.boolean().optional().describe('...'),
      })).describe('Environment variables — flag secrets with isSecret:true'),
```

- `src/tools/stacks.ts:291` — replace mode rebuilds the whole file the same
  way:

```typescript
            newContent = payloadNonSecrets.map((v) => `${v.key}=${v.value}`).join('\n');
```

- `src/utils/env-helpers.ts:82-93` — `parseDotEnvKeys` splits on `/\r?\n/`, so
  injected lines round-trip as real keys.

- `remove_stack_env_vars` and `check_stack_env_collisions` in
  `src/tools/stacks.ts` consume the same file; they take keys, not values, but
  a key with a newline has the same effect — the fix must cover keys
  everywhere too.

Conventions:

- Pure helpers live in `src/utils/env-helpers.ts` and are unit-tested in
  `tests/env-helpers.test.ts` (import style and test structure to copy).
- Tool-level validation errors: throw `new Error('...')` inside the handler —
  `registerTool`'s shared try/catch converts it to an `errorResponse`
  (see `src/utils/tool-helper.ts:70-97`).
- **Do not reformat the Zod schemas** — several tests regex-match tool source
  text (e.g. `tests/api-contracts.test.ts`, `tests/stack-env-merge.test.ts`).
  Add `.regex(...)`/`.refine(...)` inline on the existing lines without
  changing surrounding line structure more than necessary, and run the full
  suite after.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Typecheck tests | `npm run typecheck:tests` | exit 0 |
| Env-helper tests | `npx vitest run tests/env-helpers.test.ts` | all pass |
| Stack env suites | `npx vitest run tests/stack-env-merge.test.ts tests/stack-env-merge-behavior.test.ts tests/stack-env-collisions.test.ts tests/stack-env-remove.test.ts tests/stack-env-tools.test.ts` | all pass |
| Full tests | `npm test` | all pass |

## Scope

**In scope**:
- `src/utils/env-helpers.ts` — add a shared validator + defensive checks.
- `src/tools/stacks.ts` — tighten the `update_stack_env` schema
  (`variables[].key`, `variables[].value`) and validate keys in
  `remove_stack_env_vars`.
- `tests/env-helpers.test.ts` — extend.
- `tests/stack-env-newline-rejection.test.ts` (create).

**Out of scope**:
- `update_stack_env_raw` / `get_stack_env_raw` — raw-content tools whose
  entire contract is "write this text verbatim"; newlines are legitimate
  there. Do not touch.
- Quoting/escaping support for multi-line values — a design change; this plan
  rejects, it does not encode.
- `src/tools/git-stacks.ts` env-file tools — they PUT whole-file content
  (raw contract), same as above.

## Git workflow

- Branch: `advisor/003-env-newline-rejection`.
- Commit: `fix(stacks): reject newlines in env keys and values before writing .env`
  (scope `stacks` is in the allowed list; a second commit
  `test(tests): cover newline rejection in stack env tools` if you split).
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Shared validator in env-helpers

Add to `src/utils/env-helpers.ts` (exported, near the top with the other
exports):

```typescript
/**
 * .env lines are newline-delimited; a key or value containing a line
 * separator would write extra KEY=VALUE lines the caller never declared
 * (config injection) and desynchronize the add/update/preserve summary from
 * what was actually written. Reject instead of silently corrupting.
 */
export function assertNoLineBreaks(kind: 'key' | 'value', text: string): void {
  if (/[\r\n]/.test(text)) {
    throw new Error(`env ${kind} must not contain line breaks: ${JSON.stringify(text.slice(0, 60))}`);
  }
}
```

Call it defensively at the top of `upsertDotEnv` for every entry of `vars`
(both `key` and `value`), and in `removeKeysFromDotEnv` for every key, so any
future caller inherits the guard.

**Verify**: `npm run typecheck` → exit 0;
`npx vitest run tests/env-helpers.test.ts` → all pass (existing cases have no
newlines, so nothing should break).

### Step 2: Tighten the tool schemas

In `src/tools/stacks.ts` `update_stack_env` schema, change:

- `key: z.string()` → `key: z.string().regex(/^[^\s=#]+$/, 'key must not contain whitespace, "=" or "#"')`
  (keeps the existing `.describe(...)` chained after).
- `value: z.string()` → `value: z.string().regex(/^[^\r\n]*$/, 'value must not contain line breaks')`
  (keep `.describe(...)`).

In `remove_stack_env_vars` (same file), apply the same `key` regex to its
`keys` array element schema.

Check `check_stack_env_collisions` — it reads, never writes; leave unchanged.

**Verify**: `npm run typecheck` → exit 0; then run the stack env suites
command from the table → all pass. If a regex-over-source test fails on the
changed schema lines, adjust formatting to keep the patterns it scans intact
(read the failing assertion first — do not weaken the test).

### Step 3: Regression tests

Extend `tests/env-helpers.test.ts`:
- `upsertDotEnv` throws when a value contains `\n` (and when a key does).
- `removeKeysFromDotEnv` throws on a key containing `\n`.

Create `tests/stack-env-newline-rejection.test.ts` using the fake-server
harness pattern from `tests/stack-env-merge-behavior.test.ts`:
- Invoking the captured `update_stack_env` handler with
  `variables: [{ key: 'A', value: 'x\nINJECTED=1' }]` yields an error response
  (the shared try/catch converts the schema/helper throw), and the mock
  client's `put` was **never called**.
- Same for a key of `'A\nB'`.
- A legitimate value containing spaces and `#` still succeeds (values only
  reject `\r\n`; the `#` restriction applies to keys, not values).

**Verify**: `npx vitest run tests/stack-env-newline-rejection.test.ts tests/env-helpers.test.ts` → all pass; `npm test` → all pass.

## Test plan

Covered in step 3 — 5 new cases across two files, patterned on
`tests/env-helpers.test.ts` and `tests/stack-env-merge-behavior.test.ts`.

## Done criteria

- [ ] `npm run typecheck` and `npm run typecheck:tests` exit 0
- [ ] `npm test` exits 0 including the new cases
- [ ] `grep -n "assertNoLineBreaks" src/utils/env-helpers.ts` shows the
      definition plus call sites in `upsertDotEnv` and `removeKeysFromDotEnv`
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- The excerpts no longer match (drift), especially if `update_stack_env` has
  been refactored away from `upsertDotEnv`.
- Tightening the `key` regex breaks an existing test that feeds keys with
  dots or dashes (Dockhand may allow them) — if so, relax the key regex to
  only exclude `\r`, `\n`, `=`, and leading `#`, and report the discrepancy.
- Any existing stack-env test asserts that multi-line values are accepted
  (would mean the behavior is relied upon) — report, do not override.

## Maintenance notes

- If multi-line values become a real need, the correct follow-up is quoted
  encoding in `upsertDotEnv` (double-quote + escape `\n`, `"`, `\\`) plus a
  matching parser change — a deliberate format change, not a validator tweak.
- Reviewer: confirm the schema `.describe()` texts survived (they feed the
  generated body-contract report) and that the derived-docs CI gate is green.
