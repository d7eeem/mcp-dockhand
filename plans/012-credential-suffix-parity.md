# Plan 012: Apply the repo's own credential-suffix bar to every tool that clears it

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- src/openapi/description-suffixes.ts src/tools/ tests/`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P2
- **Effort**: S
- **Risk**: LOW (description text plus one guardrail test; no behavior change)
- **Depends on**: none
- **Category**: security
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

This repo already decided, in writing, when a tool needs an operator-safety
note appended to its description. `src/openapi/description-suffixes.ts` states
the bar and deliberately sets it high:

> The bar for an entry here is deliberately high. A suffix is warranted only
> when calling the tool the obvious way has a consequence a caller cannot see
> from the endpoint's own description. "This is destructive" does not
> qualify — `delete_stack` says so itself, and every MCP client already gates
> writes. What qualifies so far is exactly one thing: **arguments that carry
> credentials**, because those land in the tool-call arguments and from there
> in transcripts and logs, which is invisible at the call site and
> irreversible after.

Four secret-provider tools got the suffix. Roughly a dozen other tools take
credentials as arguments — SSH private keys, registry passwords, OIDC client
secrets, LDAP bind passwords, SMTP passwords, user passwords, license keys —
and got nothing. That is drift against the repo's own rule, and the practical
result is uneven: a caller is warned before handing over a Vault token but not
before handing over a git deploy key.

A second, related gap the existing rule did not anticipate: three tools **mint
a plaintext credential into the response**, which by Dockhand's own API
contract is shown exactly once. `create_hawser_token` and `create_auth_token`
return a token that outlives the MCP session and is not covered by any
rotation of `DOCKHAND_PASSWORD`; `enable_user_mfa` with its default
`action: 'setup'` regenerates and returns a user's TOTP enrolment secret. The
same reasoning that justifies the request-side bar — invisible at the call
site, irreversible after — applies to the response side.

## Current state

`src/openapi/description-suffixes.ts` — the mechanism and the full current
registry (the export is the last thing in the file):

```typescript
export const TOOL_DESCRIPTION_SUFFIXES: Readonly<Record<string, string>> = {
  exec_container: EXEC_RETURNS_NO_OUTPUT,
  get_stack_env_raw: RETURNS_THE_FILE_VERBATIM,
  create_secret_provider: CONFIG_CARRIES_CREDENTIALS,
  update_secret_provider: CONFIG_CARRIES_CREDENTIALS,
  test_secret_provider: CONFIG_CARRIES_CREDENTIALS,
  test_secret_provider_config: CONFIG_CARRIES_CREDENTIALS,
};
```

The existing credential constant, to match in tone and length:

```typescript
const CONFIG_CARRIES_CREDENTIALS =
  'SECURITY: the `config` object holds this provider\'s credentials (Vault token, Infisical ' +
  ... // read the full text in the file
```

Tools that clear the written bar but have no entry (verified by reading each
schema — confirm line numbers yourself, they are leads):

**Request-side (credential-carrying arguments):**

| Tool | File | Credential-shaped params |
|---|---|---|
| `create_git_credential` | `src/tools/git-stacks.ts:81` | `password`, `sshKey`, `token` |
| `update_git_credential` | `src/tools/git-stacks.ts:109` | same |
| `create_registry` / `update_registry` | `src/tools/registries.ts:20,36` | `config` documented as containing `username, password` |
| `test_registry_connection` | `src/tools/registries.ts:119` | same |
| `create_user` | `src/tools/users.ts:22` | `password` |
| `configure_oidc` | `src/tools/auth.ts:34` (verify) | `clientSecret` inside `config` |
| `configure_ldap` | `src/tools/auth.ts:62` (verify) | `bindPassword` inside `config` |
| `create_notification` / `update_notification` / `test_notification` | `src/tools/notifications.ts:20,36,60` | SMTP password inside `config` |
| `activate_license` | `src/tools/system.ts:173` (verify) | `licenseKey` |

For reference, `create_git_credential` as it exists today:

```typescript
  registerTool(server, 'create_git_credential',
    {
      name: z.string().describe('Credential name'),
      type: z.string().describe('Credential type (e.g. ssh, token, password)'),
      username: z.string().optional().describe('Username for password-based authentication'),
      password: z.string().optional().describe('Password for password-based authentication'),
      sshKey: z.string().optional().describe('Private SSH key content for SSH authentication'),
      token: z.string().optional().describe('Personal access token for token-based authentication'),
      additionalConfig: z.record(z.string(), z.unknown()).optional().describe('Additional configuration not covered by explicit parameters'),
    },
```

**Response-side (mint a plaintext credential):**

| Tool | File | What comes back |
|---|---|---|
| `create_hawser_token` | `src/tools/auth.ts:92` | plaintext Hawser token, returned only once |
| `create_auth_token` | `src/tools/auth.ts:121` | plaintext Dockhand API token, shown only once |
| `enable_user_mfa` | `src/tools/users.ts:66` | TOTP enrolment secret when `action: 'setup'` (the default) |

`enable_user_mfa` already carries a detailed comment about `action`'s
destructive regeneration behavior — read it; the suffix should complement it,
not repeat it.

Conventions:

- Suffixes are appended to the spec-derived description by
  `describeTool()` in `src/openapi/describe-tool.ts` — read it to confirm how
  the map is consumed before adding entries.
- Existing tests for this mechanism: `tests/description-overrides.test.ts`,
  `tests/tool-descriptions-derived.test.ts`,
  `tests/describe-tool.test.ts`. Read them; one of them is the right place to
  extend, or model the new test on them.
- The CI derived-docs gate regenerates documents from tool registrations —
  descriptions feed them, so **expect** `docs/coverage.md` or
  `docs/body-contract-report.md` to change. Regenerate and commit rather than
  fighting it (see step 4).

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Typecheck | `npm run typecheck` | exit 0 |
| Typecheck tests | `npm run typecheck:tests` | exit 0 |
| Full tests | `npm test` | all pass |
| Regenerate derived docs | `npm run api:coverage-doc && npm run api:tool-endpoint-map && node scripts/generate-body-contract-doc.mjs` | exit 0 |
| Derived docs committed | `git diff --exit-code -- docs/coverage.md src/openapi/tool-endpoint-map.ts docs/body-contract-report.md` | exit 0 *after* committing regenerated output |

## Scope

**In scope**:
- `src/openapi/description-suffixes.ts` — new constants and registry entries.
- `tests/description-suffix-parity.test.ts` (create) — the guardrail.
- Regenerated derived docs, if the generators change them.

**Out of scope**:
- Tool *behavior*, schemas, and parameter names — text only.
- `exec_container` and `get_stack_env_raw` entries — leave as they are.
- Adding suffixes for destructiveness — the file explicitly rejects that bar;
  structured tool annotations are the right channel and are recorded as a
  separate follow-up in `plans/README.md`.
- Omitting `create_auth_token` from the tool set entirely (a defensible option
  the audit raised) — that is a product decision, recorded as a follow-up.

## Git workflow

- Branch: `advisor/012-credential-suffixes`.
- Commits (scopes from `.commitlintrc.json`):
  - `docs(security): warn on tools whose arguments carry credentials`
  - `docs(security): warn on tools that return a one-time plaintext credential`
  - `test(tests): guard credential-suffix parity`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Read the mechanism, then add request-side constants

Read `src/openapi/description-suffixes.ts` end to end first — including the
header comment quoted above — so your wording matches its voice (direct,
specific about the consequence, tells the caller what to do).

Add two or three constants rather than one generic string, because the
concrete credential differs:

- `ARGS_CARRY_GIT_CREDENTIALS` — for `create_git_credential` /
  `update_git_credential`: names the SSH private key / token / password, notes
  they land in tool-call arguments and therefore in transcripts and logs, and
  says to prefer having the operator create the credential in Dockhand's UI
  and reference it by id.
- `CONFIG_CARRIES_SERVICE_CREDENTIALS` — for registries, OIDC/LDAP,
  notifications: same reasoning, different secret (registry password, client
  secret, bind password, SMTP password).
- `ARGS_CARRY_USER_SECRET` — for `create_user` and `activate_license`.

Keep each to two or three sentences, in the style of the existing
`CONFIG_CARRIES_CREDENTIALS`.

**Verify**: `npm run typecheck` → exit 0.

### Step 2: Add response-side constant and entries

Add `RETURNS_ONE_TIME_CREDENTIAL`, worded for the response side — e.g. that
the call returns a plaintext credential Dockhand will never show again; that
it lands in the transcript and any client-side history; that it should be
treated as compromised if it was not deliberately requested, and revoked via
the corresponding `revoke_*` tool. For `enable_user_mfa`, note specifically
that the default `action: 'setup'` **regenerates** the secret and invalidates
the user's existing enrolment.

Register all entries in `TOOL_DESCRIPTION_SUFFIXES`.

**Verify**: `npm run typecheck` → exit 0; `npm test` → all pass (some
description tests may need their expected text updated — read the failure
first; if a test asserts an exact full description for one of these tools,
update the expectation, do not weaken the assertion).

### Step 3: Guardrail test

Create `tests/description-suffix-parity.test.ts`. Model the tool-introspection
approach on `tests/tool-registration.test.ts` (read it to see how registered
tools and their schemas are enumerated).

Assert: **every registered tool whose input schema declares a parameter whose
name matches a credential-shaped pattern has an entry in
`TOOL_DESCRIPTION_SUFFIXES`.** Pattern set: `password`, `sshKey`, `token`,
`clientSecret`, `bindPassword`, `licenseKey`, `secret`. Plus an explicit
allowlist constant in the test for the known `config`-carrying tools that hide
the credential inside an untyped record (registries, OIDC/LDAP,
notifications), since a name match cannot see into `z.record`.

Be careful with false positives: `token` also appears as a *TOTP code* param
(`enable_user_mfa`) and as `tokenId` on revoke tools. Exclude `*Id` suffixes,
and let `enable_user_mfa` match (it needs an entry anyway).

The test must fail if a future tool adds a credential parameter without a
suffix. Include a comment saying that is its whole purpose.

**Verify**: `npx vitest run tests/description-suffix-parity.test.ts` → passes;
deliberately break it once (temporarily remove one entry) to confirm it fails,
then restore.

### Step 4: Regenerate derived docs

Descriptions feed the generated documents. Run the regeneration command, then
inspect the diff: it should show only description-text changes for the tools
you touched. Commit the regenerated files.

**Verify**: after committing, `git diff --exit-code -- docs/coverage.md src/openapi/tool-endpoint-map.ts docs/body-contract-report.md` → exit 0.

## Test plan

- New: `tests/description-suffix-parity.test.ts` (the guardrail above).
- Existing description tests must stay green; update expected strings only
  where a suffix you added legitimately changed them.
- Verification: `npm test` → all pass.

## Done criteria

- [ ] `npm run typecheck` and `npm run typecheck:tests` exit 0
- [ ] `npm test` exits 0 including the new guardrail test
- [ ] `TOOL_DESCRIPTION_SUFFIXES` contains entries for all tools named in the
      two tables (confirmed present by `grep -c ":" ` on the export block, or
      by reading it)
- [ ] The guardrail test demonstrably fails when an entry is removed (state in
      your report that you verified this)
- [ ] Derived docs regenerated and committed; the gate command exits 0
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- A tool named in the tables does not exist or does not take the credential
  parameter described (drift) — skip it, note it, continue.
- `describeTool()` turns out not to append suffixes the way the file's header
  describes — read it and report before adding a dozen entries that do nothing.
- The guardrail test flags more than ~20 tools — your pattern is too broad;
  narrow it and report the list rather than adding suffixes wholesale.
- A description test asserts an exact full string for a tool you are changing
  and the change looks like it would break a *contract* rather than an
  expectation — report it.

## Maintenance notes

- The guardrail test is the durable part: suffix text can be improved freely,
  but the parity check is what stops the next credential-carrying tool from
  shipping unmarked.
- If tool annotations land later (recorded follow-up), the destructive/
  read-only signal moves to that structured channel; these suffixes stay,
  because they describe a *credential-handling* consequence that annotations
  do not express.
- Reviewer: check the wording says what the caller should DO, not just that
  something is sensitive — the existing `CONFIG_CARRIES_CREDENTIALS` is the
  quality bar.
