# Plan 008: Harden the fork-sync pipeline — gate before executing upstream code, and shrink its privileges

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- Jenkinsfile scripts/sync-upstream.sh`
> If either file changed since this plan was written, compare the "Current
> state" excerpts against the live code before proceeding; on a mismatch,
> treat it as a STOP condition.

## Status

- **Priority**: P1
- **Effort**: M
- **Risk**: MED (touches the working push path; the pipeline cannot be fully
  exercised from the repo — see "Verification reality" below)
- **Depends on**: none
- **Category**: security
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

`Jenkinsfile` + `scripts/sync-upstream.sh` are the entire reason the
`hardened` branch exists: they rebase this fork onto each new upstream release
and gate it before adoption. Five problems, all verified by reading the file:

1. **The gates run after the untrusted code executes.** Stage order is
   *Sync + rebase* → **Install, build, test** → *dependency audit* → *gitleaks*
   → *trivy fs*. The rebase has just pulled a freshly published upstream tag,
   and the very next stage runs `npm ci` (executing install lifecycle scripts
   from the whole dependency tree) and `npm test` — on the agent that holds a
   Contents:RW PAT for the fork and a Docker socket. A compromised or
   malicious upstream release is therefore **executed first and scanned
   second**. That inverts the pipeline's stated purpose.
2. **A third-party `:latest` image gets the Docker socket.** The image scan
   runs `docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest`
   — root-equivalent control of the build agent, granted to a mutable tag.
3. **The PAT is passed as a command-line argument.** `PUSH_URL` embeds
   `${GIT_USER}:${GIT_TOKEN}` and is passed to `git push`, so the credential
   is visible in the agent's process table for the duration of the push.
   (Jenkins masks console output — that is a different exposure than argv.)
4. **`npm audit` gates on the dev tree.** The shipped image installs
   `--omit=dev`; production currently has **0** vulnerabilities while the dev
   tree has 7 advisories (2 HIGH) inside the npm copy bundled in
   `semantic-release`. `npm audit --audit-level=high` therefore fails **today,
   on every run**, for packages that never reach the artifact — a red gate
   with no defect, which is how gates get ignored or disabled.
5. **The pipeline can't run as committed.** `FORK_REPO` is the literal
   placeholder `YOUR_GH_USER`, and both the header and the failure path point
   at `fork-kit/README.md`, which does not exist in this repo. Fixing the
   security posture now — before it is wired up — is the cheap moment.

## Current state

`Jenkinsfile:34-40` (environment block):

```groovy
    environment {
        FORK_REPO       = 'github.com/YOUR_GH_USER/mcp-dockhand.git'   // <-- EDIT
        UPSTREAM_REPO   = 'https://github.com/strausmann/mcp-dockhand.git'
        HARDENED_BRANCH = 'hardened'
        MIRROR_BRANCH   = 'main'
```

Stage order (`grep -n "stage(" Jenkinsfile`):

```
47:  stage('Checkout fork')
72:  stage('Detect upstream release')
90:  stage('Sync mirror + rebase hardened')
99:  stage('Install, build, test')          <-- executes upstream code
111: stage('Security: dependency audit')    <-- gates run after
120: stage('Security: secret scan (gitleaks)')
134: stage('Security: filesystem scan (trivy)')
149: stage('Docker image build + scan')
171: stage('Push secured branch to fork')
```

`Jenkinsfile:99-110`:

```groovy
        stage('Install, build, test') {
            when { expression { env.NEW_TAG?.trim() } }
            agent { docker { image 'node:22-bookworm'; reuseNode true } }
            steps {
                sh '''
                    npm ci
                    npm run build --if-present
                    npm test --if-present
                '''
            }
        }
```

`Jenkinsfile:111-118`:

```groovy
        stage('Security: dependency audit') {
            when { expression { env.NEW_TAG?.trim() } }
            agent { docker { image 'node:22-bookworm'; reuseNode true } }
            steps {
                // Gate on high/critical dependency CVEs.
                sh 'npm audit --audit-level=high'
            }
        }
```

`Jenkinsfile:149-170` (image build + scan, the socket mount):

```groovy
                sh '''
                    docker build -t "mcp-dockhand:hardened-$NEW_TAG" .
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v trivy-cache:/root/.cache/trivy \
                      aquasec/trivy:latest image \
                      --exit-code 1 --severity HIGH,CRITICAL \
                      "mcp-dockhand:hardened-$NEW_TAG"
                '''
```

`Jenkinsfile:180-190` (push, credential in URL):

```sh
                    set -euo pipefail
                    PUSH_URL="https://${GIT_USER}:${GIT_TOKEN}@${FORK_REPO}"
                    git push "$PUSH_URL" "$MIRROR_BRANCH:$MIRROR_BRANCH"
                    git push --force-with-lease "$PUSH_URL" "$HARDENED_BRANCH:$HARDENED_BRANCH"
                    git tag -f "hardened-$NEW_TAG" "$HARDENED_BRANCH"
                    git push -f "$PUSH_URL" "refs/tags/hardened-$NEW_TAG"
                    git push "$PUSH_URL" --tags || true   # mirror upstream v* tags, best effort
```

`scripts/sync-upstream.sh:117-125`:

```sh
cmd_verify() {
  [ -f package.json ] || die "no package.json here"
  npm ci
  npm audit --audit-level=high
  npm run build --if-present
  npm test --if-present
  log "verify passed"
}
```

Other verified facts:

- `Dockerfile:33` installs `npm ci --omit=dev --ignore-scripts` — the shipped
  tree is dev-free, which is what makes item 4 above a scoping bug.
- Scanner images `zricethezav/gitleaks:latest` (`Jenkinsfile:124`) and
  `aquasec/trivy:latest` (`:138`, `:162`) are mutable tags; the repo pins
  precisely elsewhere (`scripts/lint-in-container.sh:27` pins three npm
  packages by exact version, with a comment explaining why).
- `scripts/sync-upstream.sh` exits 3 on a real rebase conflict (fails closed) —
  that behavior is good; do not change it.

## Verification reality (read before starting)

You **cannot** run this pipeline from the repo — it needs a Jenkins
controller, a `docker`-labelled agent, and a `github-pat` credential. Your
verification is therefore limited to:

- **Syntax**: the `Jenkinsfile` must remain valid Declarative Pipeline.
  If a Jenkins CLI is unavailable (expect it to be), verify by careful
  reading: balanced braces, every `stage` inside `stages`, `agent`/`when`/
  `steps` in valid positions. Do not introduce Groovy string interpolation of
  any credential — keep credential use inside single-quoted `sh ''' ... '''`
  blocks so the value is expanded by the shell, never by Groovy (the existing
  code comments call this out at `Jenkinsfile:179`).
- **Shell**: `bash -n scripts/sync-upstream.sh` must pass, and
  `shellcheck scripts/sync-upstream.sh` if shellcheck is installed (optional).
- **Repo suite**: `npm test` must stay green (no source changes expected).

State plainly in your final report that the pipeline itself was not executed.

## Scope

**In scope**:
- `Jenkinsfile`
- `scripts/sync-upstream.sh` (the `cmd_verify` audit scope only)
- `docs/fork-pipeline.md` (create — the missing prereq/recovery doc that the
  Jenkinsfile header and failure path currently point at as
  `fork-kit/README.md`)

**Out of scope**:
- `.github/workflows/*` — GitHub Actions hardening is plan 004 and a separate
  recorded follow-up. Do not touch them here.
- The rebase/rerere logic in `scripts/sync-upstream.sh` (`cmd_sync`,
  `cmd_detect`) — it fails closed and is working as designed.
- Adding a `docker login`/`docker push` to the image stage — deliberately left
  as an operator TODO in the current file; keep it that way.
- Making the GitHub Actions CI and this pipeline share one gate definition —
  a recorded follow-up (DX-05), larger than this plan.

## Git workflow

- Branch: `advisor/008-jenkins-hardening`.
- Commits (scopes from `.commitlintrc.json` — `ci`, `security`, `docs`):
  - `ci(security): run fork-sync gates before executing upstream code`
  - `ci(security): drop the docker socket mount and pin scanner images`
  - `ci(security): pass the fork PAT via stdin instead of the push URL`
  - `ci(deps): scope the fork-sync npm audit to production dependencies`
  - `docs(ci): document the fork-sync pipeline prereqs and recovery`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Reorder — gates before code execution

Move the three security stages (`Security: dependency audit`,
`Security: secret scan (gitleaks)`, `Security: filesystem scan (trivy)`) so
they run **immediately after** `Sync mirror + rebase hardened` and **before**
`Install, build, test`. Keep each stage's `when`, `agent` and `steps` blocks
otherwise intact at this step.

The dependency audit needs a dependency tree to audit but must not execute
lifecycle scripts. Change its step to install without scripts first:

```groovy
            steps {
                // --ignore-scripts: this stage runs BEFORE 'Install, build, test',
                // on code that was just rebased onto a fresh upstream tag. Gating
                // must not be the thing that first executes that code.
                sh '''
                    npm ci --ignore-scripts --no-audit --no-fund
                    npm audit --omit=dev --audit-level=high
                '''
            }
```

(The `--omit=dev` part is step 4; writing it here at once is fine — just make
sure the commit split matches the messages above, or squash them into one
`ci(security):` commit and say so in the report.)

Add a short comment above the reordered block recording *why* the order
matters, in the style the repo uses elsewhere (`.github/workflows/ci.yml` has
several such explanatory comments):

```groovy
        // Gates run BEFORE 'Install, build, test'. The preceding stage rebases
        // onto a freshly published upstream tag, so the first thing that
        // executes that code must not be this agent — it holds a Contents:RW
        // PAT for the fork. npm ci in the audit stage uses --ignore-scripts
        // for the same reason.
```

**Verify**: read the file top to bottom and confirm stage order is
Checkout → Detect → Sync+rebase → audit → gitleaks → trivy fs →
Install/build/test → Docker build+scan → Push; braces balanced.

### Step 2: Pin the scanner images

Replace the three floating tags with pinned references:

- `zricethezav/gitleaks:latest` → `ghcr.io/gitleaks/gitleaks:v8.30.0`
  (gitleaks' current publishing location; the Docker Hub path is legacy).
- `aquasec/trivy:latest` (both sites) → `aquasec/trivy:0.60.0`.

Use whatever the current stable versions are at execution time — check
`https://github.com/gitleaks/gitleaks/releases` and
`https://github.com/aquasecurity/trivy/releases` if you have network access;
if you do not, use the versions above and note in your report that they should
be confirmed. Add a comment noting these are pinned deliberately and must be
bumped intentionally (mirroring `scripts/lint-in-container.sh`'s comment
style).

**Verify**: `grep -n ":latest" Jenkinsfile` → no matches.

### Step 3: Remove the Docker socket mount

Replace the socket-mounting image scan with a tarball-based scan, so the
scanner never gets control of the daemon:

```groovy
                sh '''
                    docker build -t "mcp-dockhand:hardened-$NEW_TAG" .
                    docker save "mcp-dockhand:hardened-$NEW_TAG" -o image.tar
                    docker run --rm \
                      -v "$PWD":/work -w /work \
                      -v trivy-cache:/root/.cache/trivy \
                      aquasec/trivy:0.60.0 image \
                      --input image.tar \
                      --exit-code 1 --severity HIGH,CRITICAL
                    rm -f image.tar
                '''
```

Keep the `trivy-cache` volume (it is the pipeline's own cache, not a
privilege). Ensure `image.tar` is removed even on scan failure — either add it
to a `post { always { sh 'rm -f image.tar' } }` block on that stage, or accept
the workspace cleanup Jenkins already performs; prefer the explicit `post`
block.

**Verify**: `grep -n "docker.sock" Jenkinsfile` → no matches.

### Step 4: Scope the audits to what ships

In `Jenkinsfile`, the audit step already gained `--omit=dev` in step 1. Do the
same in `scripts/sync-upstream.sh` `cmd_verify`:

```sh
  npm ci
  npm audit --omit=dev --audit-level=high
```

Add a one-line comment in both places: the shipped image installs
`--omit=dev --ignore-scripts` (`Dockerfile:33`), so a dev-tree advisory is not
an artifact vulnerability; gating on it turns the pipeline red without a
defect.

Optionally add a **non-gating** dev-tree visibility line right after
(`npm audit --audit-level=critical || true`) so dev advisories are still
printed. If you add it, make sure it cannot fail the stage.

**Verify**: `bash -n scripts/sync-upstream.sh` → exit 0;
`grep -n "omit=dev" Jenkinsfile scripts/sync-upstream.sh` → both present.

### Step 5: Keep the PAT out of the process table

Replace the credential-in-URL push with a stdin-fed credential helper. Inside
the same single-quoted `sh ''' ... '''` block (so Groovy never sees the
value):

```sh
                    set -euo pipefail
                    # The PAT must not appear in argv (visible in the agent's
                    # process table). Feed it to git's credential store on stdin
                    # and push to the plain https URL instead.
                    git config --local credential.helper ''
                    git config --local --add credential.helper 'store --file=.git/fork-credentials'
                    printf 'protocol=https\nhost=github.com\nusername=%s\npassword=%s\n\n' \
                      "$GIT_USER" "$GIT_TOKEN" | git credential-store --file=.git/fork-credentials store
                    PUSH_URL="https://${FORK_REPO}"
                    git push "$PUSH_URL" "$MIRROR_BRANCH:$MIRROR_BRANCH"
                    git push --force-with-lease "$PUSH_URL" "$HARDENED_BRANCH:$HARDENED_BRANCH"
                    git tag -f "hardened-$NEW_TAG" "$HARDENED_BRANCH"
                    git push -f "$PUSH_URL" "refs/tags/hardened-$NEW_TAG"
                    git push "$PUSH_URL" --tags || true
```

Add a `post { always { sh 'rm -f .git/fork-credentials; git config --local --unset-all credential.helper || true' } }`
on this stage so the credential file never survives the build. Note
`FORK_REPO` currently starts with `github.com/...` (no scheme) — keep that
shape so `https://${FORK_REPO}` composes correctly, and confirm the host in
the `printf` matches `FORK_REPO`'s host.

**Verify**: `grep -n 'GIT_TOKEN}@' Jenkinsfile` → no matches;
`grep -c "fork-credentials" Jenkinsfile` → at least 3 (store, use, cleanup).

### Step 6: Parameterize the placeholder and write the missing doc

- Replace the hardcoded `FORK_REPO` placeholder with a Jenkins job parameter
  so an unedited checkout cannot silently push to a nonexistent repo. Add a
  `parameters { string(name: 'FORK_REPO', defaultValue: '', description: 'host/owner/repo.git for the fork, e.g. github.com/you/mcp-dockhand.git') }`
  block and reference `params.FORK_REPO` in `environment`, plus a fail-fast
  check in the first stage: if it is empty, `error 'FORK_REPO parameter is not set'`.
- Create `docs/fork-pipeline.md` covering: the credential id (`github-pat`,
  fine-grained PAT with Contents: RW on the fork — **name the credential type
  only, never a value**), the required `docker` agent label, the
  `FORK_REPO` parameter, the gate order and why, and the conflict-recovery
  path (`scripts/sync-upstream.sh` exits 3; how to resolve and re-run, and
  that rerere is in play).
- Update the two `fork-kit/README.md` references in `Jenkinsfile` (header
  comment around line 13, and the conflict note around line 199) to point at
  `docs/fork-pipeline.md`.

**Verify**: `grep -rn "fork-kit" Jenkinsfile` → no matches;
`grep -n "YOUR_GH_USER" Jenkinsfile` → no matches;
`test -f docs/fork-pipeline.md` → exists.

## Test plan

There is no automated test surface for a Jenkinsfile in this repo, and adding
a Jenkins harness is out of proportion. Verification is the per-step `grep`/
syntax checks above plus:

- `bash -n scripts/sync-upstream.sh` → exit 0.
- `npm test` → still green (no source files change in this plan).
- A careful read-through confirming Declarative Pipeline structure.

If the repo later gains a lint for CI definitions, this plan's changes should
be its first customer.

## Done criteria

- [ ] Stage order: gates precede `Install, build, test` (verified by reading
      `grep -n "stage(" Jenkinsfile` output in order)
- [ ] `grep -n ":latest" Jenkinsfile` → no matches
- [ ] `grep -n "docker.sock" Jenkinsfile` → no matches
- [ ] `grep -n 'GIT_TOKEN}@' Jenkinsfile` → no matches
- [ ] `grep -n "omit=dev" Jenkinsfile scripts/sync-upstream.sh` → present in both
- [ ] `grep -n "YOUR_GH_USER\|fork-kit" Jenkinsfile` → no matches
- [ ] `docs/fork-pipeline.md` exists and names no credential values
- [ ] `bash -n scripts/sync-upstream.sh` exits 0; `npm test` exits 0
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated, noting the pipeline was not executed

## STOP conditions

- The Jenkinsfile has been substantially rewritten since `3d55424` (drift) —
  re-read it fully and report before applying any step.
- `git credential-store` is unavailable in the push stage's environment, or
  the agent's git is too old for `--force-with-lease` semantics you rely on —
  report; do NOT fall back to the credential-in-URL form.
- Reordering the stages turns out to break a `when { expression { env.NEW_TAG } }`
  dependency (e.g. a stage sets a variable a gate needs) — trace the variable,
  report, and do not silently drop a gate.
- You cannot determine current stable scanner versions and have no network —
  pin to the versions named in step 2 and flag them for confirmation.

## Maintenance notes

- Pinned scanner versions now need deliberate bumps. The repo's Renovate
  config does not track Groovy `docker { image ... }` lines — a recorded
  follow-up in `plans/README.md` is to add a `customManagers.regex` entry
  covering both `Jenkinsfile` and `scripts/lint-in-container.sh`.
- The `--ignore-scripts` install in the audit stage means that stage's
  `node_modules` is not necessarily identical to the build stage's. That is
  intentional (audit reads the tree, it does not run it), but if a future
  change makes the audit stage depend on built artifacts, it must not simply
  drop the flag.
- Reviewer should scrutinize: that no credential value can reach Groovy string
  interpolation, that the `post` cleanup blocks run on failure paths too, and
  that the gate reordering did not accidentally place a gate after the push.
- Related follow-up (not in this plan): the GitHub Actions CI and this
  pipeline enforce different gate sets; a shared `npm run ci:verify` would
  define them once.
