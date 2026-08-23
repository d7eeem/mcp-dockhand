# Plan 004: Make the documented hardening actually deployable, and close two CI supply-chain gaps

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- docker-compose.yml .env.example README.md .github/workflows/ci.yml .github/workflows/release.yml`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: S–M (four independent small fixes; each step is standalone)
- **Risk**: LOW
- **Depends on**: none
- **Category**: security
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

Four verified gaps, all cheap to fix:

1. **The shipped `docker-compose.yml` silently drops every hardening env
   var.** Compose only passes variables listed under `environment:`; the file
   lists just `DOCKHAND_*` + `MCP_PORT`. An operator who follows the README's
   "Securing the transport" instructions and sets `MCP_AUTH_TOKEN` /
   `MCP_ALLOWED_HOSTS` in `.env` gets **no protection at all** on the primary
   documented deployment path — the variables never reach the container.
2. **`.env.example` publishes a real-looking internal hostname and account
   name** (only `DOCKHAND_PASSWORD` uses a `CHANGE_ME` placeholder). In a
   public repo that is a targeting hint plus half a credential pair.
3. **`ci.yml` has no `permissions:` block** and runs `npm ci` + tests on
   `pull_request` checkouts — the only workflow in the repo without explicit
   least-privilege scopes.
4. **The release job — the highest-privilege job (pushes GHCR images, commits
   to main) — runs `npm install` on `node lts/*`** instead of `npm ci` on
   Node 22: it re-resolves dependency ranges no CI job validated and commits
   the rewritten lockfile via semantic-release, defeating the
   `lockfile-integrity` gate that exists because of a real past incident
   (v1.8.0–1.8.2 shipped no image).

## Current state

- `docker-compose.yml:8-12`:

```yaml
    environment:
      - DOCKHAND_URL=${DOCKHAND_URL}
      - DOCKHAND_USERNAME=${DOCKHAND_USERNAME}
      - DOCKHAND_PASSWORD=${DOCKHAND_PASSWORD}
      - MCP_PORT=8080
```

- `.env.example:4` and `:7` contain concrete host/username values (do NOT
  copy them anywhere; replace them). `:8` is `DOCKHAND_PASSWORD=CHANGE_ME` —
  the pattern the other two should follow. The README's own quick-start
  (`README.md:29-31`) already uses the right placeholders:
  `https://your-dockhand-server.com`, `your-username`.

- `.github/workflows/ci.yml:1-8` — `name: CI`, `on: push/pull_request`, then
  straight into `jobs:`; no `permissions:` key anywhere in the file. The other
  three workflows all declare one (`release.yml:22-27`,
  `api-schema-sync.yml:8-11`, `renovate.yml:9-10`).

- `.github/workflows/release.yml:63-70`:

```yaml
      - name: Setup Node.js
        if: ${{ github.event.inputs.build_tag == '' }}
        uses: actions/setup-node@v7
        with:
          node-version: "lts/*"

      - name: Install dependencies
        if: ${{ github.event.inputs.build_tag == '' }}
        run: npm install
```

  Every other install site in the repo uses `npm ci`
  (`Dockerfile:4`, `Dockerfile:33`, `ci.yml:27`, `ci.yml:102`,
  `api-schema-sync.yml:31`, `scripts/sync-upstream.sh:119`), and every other
  job pins Node 22. `.releaserc.json` lists `package-lock.json` in the
  `@semantic-release/git` assets, so any lockfile mutation here is committed
  onto the release tag that `docker-build` then checks out.

- README config table (`README.md:64-78`) documents all the env vars; the
  compose snippet at `README.md:38-50` has the same omission as the shipped
  compose file.

- `tests/healthcheck-loopback.test.ts` regex-checks `docker-compose.yml` text
  — after editing compose, run it.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Compose file validity | `docker compose -f docker-compose.yml config -q` (if Docker present; else `npx yaml docker-compose.yml` or any YAML parse) | exit 0 |
| Workflow YAML validity | `node -e "const y=require('js-yaml');const fs=require('fs');for(const f of ['.github/workflows/ci.yml','.github/workflows/release.yml'])y.load(fs.readFileSync(f,'utf8'));console.log('ok')"` (js-yaml is in node_modules via transitive deps; if unavailable, use any YAML linter) | prints ok |
| Affected tests | `npx vitest run tests/healthcheck-loopback.test.ts` | all pass |
| Full tests | `npm test` | all pass |

## Scope

**In scope**:
- `docker-compose.yml`
- `.env.example`
- `README.md` (compose snippet + a one-line note; no other README surgery)
- `.github/workflows/ci.yml` (permissions block only)
- `.github/workflows/release.yml` (two lines: node-version, npm ci)

**Out of scope**:
- Changing any security **default** in code (`MCP_HOST`, opt-in guards,
  `MCP_MAX_SESSIONS`) — those are documented deliberate decisions; flipping
  them is the separate "hardened-fork identity" discussion.
- The Jenkinsfile / `scripts/sync-upstream.sh` (its own follow-up; see
  plans/README.md).
- Pinning GitHub Actions to SHAs (recorded as a follow-up; touching every
  workflow line is a separate mechanical PR best done with Renovate's
  `pinDigests`).
- Rotating the Dockhand account password — **you cannot do this from the
  repo**; it is called out in the Done criteria as an operator action to
  flag, not perform.

## Git workflow

- Branch: `advisor/004-deploy-ci-hardening`.
- Commits (scopes from `.commitlintrc.json`):
  - `fix(security): pass transport-hardening env vars through docker-compose`
  - `fix(security): replace real host and account name in .env.example with placeholders`
  - `ci(ci): add least-privilege permissions block to ci.yml`
  - `ci(release): install with npm ci on Node 22 in the release job`
  (type `ci` requires a scope too — `ci` and `release` are both in the
  scope-enum.)
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Compose pass-through

In `docker-compose.yml`, extend the `environment:` list (keep the existing
four lines first, preserve comment style):

```yaml
    environment:
      - DOCKHAND_URL=${DOCKHAND_URL}
      - DOCKHAND_USERNAME=${DOCKHAND_USERNAME}
      - DOCKHAND_PASSWORD=${DOCKHAND_PASSWORD}
      - MCP_PORT=8080
      # Transport hardening (README "Securing the transport") — empty when
      # unset in .env, which the server treats as "check disabled".
      - MCP_ALLOWED_HOSTS=${MCP_ALLOWED_HOSTS:-}
      - MCP_ALLOWED_ORIGINS=${MCP_ALLOWED_ORIGINS:-}
      - MCP_AUTH_TOKEN=${MCP_AUTH_TOKEN:-}
      - TRUSTED_PROXIES=${TRUSTED_PROXIES:-}
      - LOG_LEVEL=${LOG_LEVEL:-info}
      - MCP_SESSION_TTL_SECONDS=${MCP_SESSION_TTL_SECONDS:-}
      - MCP_SESSION_CLEANUP_INTERVAL_SECONDS=${MCP_SESSION_CLEANUP_INTERVAL_SECONDS:-}
      - MCP_MAX_SESSIONS=${MCP_MAX_SESSIONS:-}
```

Empty-string values must behave like unset for every one of these — verify by
reading the parsers: `getTransportSecurityConfig`
(`src/auth/transport-guard.ts:78-88`) and `getSessionLifecycleConfig`
(`src/session-lifecycle.ts:38-48` — `value.trim() === ''` → fallback) treat
`''` as disabled/default. Confirm `TRUSTED_PROXIES` and `LOG_LEVEL` parsing do
the same (`src/server.ts:76`, `src/utils/logger.ts`); if any parser
distinguishes empty from unset, STOP and report which.

Mirror the same block into the README compose snippet (`README.md:38-50`).

**Verify**: compose config command → exit 0;
`npx vitest run tests/healthcheck-loopback.test.ts` → all pass.

### Step 2: Neutral placeholders in `.env.example`

Replace the values of `DOCKHAND_URL` (line 4) and `DOCKHAND_USERNAME`
(line 7) with the README's placeholder forms:
`https://your-dockhand-server.com` and `your-username`. Do not echo the old
values into the commit message, the diff description, or anywhere else — they
are identifier disclosure.

**Verify**: `grep -nE "your-dockhand-server|your-username" .env.example` →
both lines present; `grep -cE "CHANGE_ME" .env.example` → unchanged count.

### Step 3: ci.yml permissions

Add at workflow level (after `name: CI`, before `on:` or after it — match the
placement style of `api-schema-sync.yml:8-11`):

```yaml
permissions:
  contents: read
```

No job in `ci.yml` writes to the repo, packages, or PRs (jobs: build, docker
smoke `push: false`, lockfile-integrity, lint) — `contents: read` suffices.

**Verify**: workflow YAML validity command → ok.

### Step 4: release.yml install hygiene

In `.github/workflows/release.yml`:
- Line 65: `node-version: "lts/*"` → `node-version: 22` (matching `ci.yml`'s
  matrix and the `node:22-alpine` runtime).
- Line 69: `run: npm install` → `run: npm ci`.

Do not touch anything else in the file (it also contains the docker-build and
merge jobs with their own permissions).

**Verify**: workflow YAML validity command → ok; `grep -n "npm install" .github/workflows/release.yml` → no matches.

## Test plan

No new unit tests (config-only changes). Existing gates:
`tests/healthcheck-loopback.test.ts` (compose text), full `npm test` for
regressions. The real verification of step 4 happens on the next release run —
note that in the PR description.

## Done criteria

- [ ] All four verify commands pass
- [ ] `npm test` exits 0
- [ ] `grep -n "MCP_AUTH_TOKEN" docker-compose.yml` → present
- [ ] `.env.example` contains no hostname other than the placeholder and no
      username other than `your-username`
- [ ] `.github/workflows/ci.yml` contains `permissions:` with
      `contents: read`
- [ ] `.github/workflows/release.yml` uses `npm ci` and `node-version: 22`
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated, including this note verbatim:
      "Operator action required: the Dockhand account name previously in
      .env.example has been public — rotate that account's password."

## STOP conditions

- Any parser treats empty-string env vars differently from unset (step 1
  check) — report which variable before shipping the compose change.
- `release.yml`'s `npm ci` premise breaks locally: run `npm ci` once in a
  scratch clone (or rely on the existing `lockfile-integrity` CI job); if the
  committed lockfile cannot satisfy `npm ci`, the lockfile is stale — report,
  do not "fix" it by keeping `npm install`.
- The healthcheck-loopback test asserts an exact `environment:` block shape
  that the new lines break — read the test, extend the assertion minimally,
  and note it in the commit.

## Maintenance notes

- Follow-ups deliberately not in this plan (see plans/README.md): pinning
  third-party actions by SHA (Renovate `pinDigests`), narrowing the Infisical
  `secret-path: /` in `renovate.yml`, Jenkinsfile gate reordering, an
  `env_file:`-based compose alternative.
- Reviewer: confirm no real hostname/username appears in the diff contexts
  GitHub shows (the removed lines will show them — that is unavoidable and
  fine, they are already public in history; rotation is the remedy).
- If a future env var is added to the README table, it must also be added to
  the compose `environment:` list — consider a follow-up test asserting
  parity between the README table and docker-compose.yml.
