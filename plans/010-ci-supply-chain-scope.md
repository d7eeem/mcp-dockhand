# Plan 010: Shrink the CI supply-chain blast radius — narrow the secret pull, pin actions by digest

> **Executor instructions**: Follow this plan step by step. Run every
> verification command and confirm the expected result before moving to the
> next step. If anything in the "STOP conditions" section occurs, stop and
> report — do not improvise. When done, update the status row for this plan
> in `plans/README.md` — unless a reviewer dispatched you and told you they
> maintain the index.
>
> **Drift check (run first)**: `git diff --stat 3d55424..HEAD -- .github/workflows/ renovate.json`
> If any in-scope file changed since this plan was written, compare the
> "Current state" excerpts against the live code before proceeding; on a
> mismatch, treat it as a STOP condition.

## Status

- **Priority**: P2
- **Effort**: S–M
- **Risk**: LOW–MED (digest pinning changes PR volume; the secret-path change
  can break the Renovate job if the path is wrong)
- **Depends on**: plan 004 should land first — it adds the `permissions:`
  block to `ci.yml` and both plans touch workflow files (avoid conflicts)
- **Category**: security
- **Planned at**: commit `3d55424`, 2026-08-20

## Why this matters

Two supply-chain gaps in the GitHub Actions setup:

1. **The Renovate job pulls every secret in an Infisical project/environment
   into the job environment** (`secret-path: /`) and then immediately runs a
   third-party action in that populated environment. It needs exactly one
   value — `RENOVATE_TOKEN`. The blast radius of any compromise in the next
   step is currently "every secret under that path in the prod environment",
   not "the Renovate token".

2. **All 13 third-party action references are mutable version tags.** A tag
   can be re-pointed by its maintainer or by anyone who compromises that
   repository, and the change is invisible in this repo's diff. This matters
   most in `release.yml`, whose jobs hold `packages: write` and push images to
   ghcr.io, and in the Renovate job described above. Meanwhile `renovate.json`
   already contains a rule matching `"pin"` and `"digest"` update types —
   but `pinDigests` is never enabled, so Renovate never emits those update
   types and the rule is **inert**: the config reads as if digests are managed
   when nothing is.

## Current state

`.github/workflows/renovate.yml` (the whole job — note `permissions:` is
already correctly scoped here):

```yaml
permissions:
  contents: read
jobs:
  renovate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - name: RENOVATE_TOKEN aus Infisical holen
        uses: Infisical/secrets-action@v1.0.16
        with:
          method: universal
          client-id: ${{ secrets.INFISICAL_CLIENT_ID }}
          client-secret: ${{ secrets.INFISICAL_CLIENT_SECRET }}
          project-slug: fileee-server-ci
          env-slug: prod
          secret-path: /
          domain: https://secretsmanager.strausmann.cloud
      - uses: renovatebot/github-action@v46.2.1
        with:
          token: ${{ env.RENOVATE_TOKEN }}
          configurationFile: renovate.json
        env:
          RENOVATE_REPOSITORIES: ${{ github.repository }}
```

The complete set of third-party action references
(`grep -rn "uses:" .github/workflows/*.yml`):

```
renovate.yml:17          Infisical/secrets-action@v1.0.16
renovate.yml:26          renovatebot/github-action@v46.2.1
ci.yml:79                docker/setup-buildx-action@v4
ci.yml:82                docker/build-push-action@v7
api-schema-sync.yml:135  actions/github-script@v9
api-schema-sync.yml:172  actions/upload-artifact@v7
release.yml:147          docker/setup-buildx-action@v4
release.yml:150          docker/login-action@v4
release.yml:162          docker/build-push-action@v7
release.yml:192          actions/upload-artifact@v7
release.yml:209          actions/download-artifact@v8
release.yml:224          docker/setup-buildx-action@v4
release.yml:227          docker/login-action@v4
```

Plus `actions/checkout@v7` and `actions/setup-node@v7` at several sites.

`renovate.json` — the inert rule is the first `packageRules` entry:

```json
  "packageRules": [
    {
      "matchUpdateTypes": ["minor", "patch", "pin", "digest"],
      "automerge": true
    },
```

There is no `"pinDigests": true` anywhere in the file, and `extends` is
`config:recommended`, `:semanticCommits`, `:enableVulnerabilityAlerts` — none
of which enable digest pinning.

## Commands you will need

| Purpose | Command | Expected on success |
|---|---|---|
| Workflow YAML parses | `node -e "const y=require('js-yaml'),fs=require('fs');for(const f of require('fs').readdirSync('.github/workflows'))y.load(fs.readFileSync('.github/workflows/'+f,'utf8'));console.log('ok')"` | prints `ok` |
| Renovate config parses | `node -e "JSON.parse(require('fs').readFileSync('renovate.json','utf8'));console.log('ok')"` | prints `ok` |
| Full tests | `npm test` | all pass (no source change expected) |

If `js-yaml` is not resolvable, use any available YAML parser or
`python3 -c "import yaml,sys,glob; [yaml.safe_load(open(f)) for f in glob.glob('.github/workflows/*.yml')]; print('ok')"`.

## Scope

**In scope**:
- `.github/workflows/renovate.yml` — narrow `secret-path`.
- `renovate.json` — enable `pinDigests` (or remove the inert rule; see step 3).
- All `.github/workflows/*.yml` — SHA-pin third-party actions.

**Out of scope**:
- `ci.yml`'s `permissions:` block and `release.yml`'s `npm ci`/node version —
  those are plan 004. If plan 004 has not landed, do not do its work here.
- `Jenkinsfile` scanner pinning — plan 008.
- Adding a Renovate `customManagers` regex for `Jenkinsfile` /
  `scripts/lint-in-container.sh` — recorded as a follow-up; it is a different
  mechanism from digest pinning and would collide with plan 008's edits.
- `actions/*` first-party actions may be pinned too, but if you want to limit
  churn, pinning the **third-party** ones (Infisical, renovatebot, docker/*)
  is the security-relevant subset. Decide once and be consistent; state which
  you chose.

## Git workflow

- Branch: `advisor/010-ci-supply-chain`.
- Commits (scopes from `.commitlintrc.json`):
  - `ci(security): pull only the Renovate token from Infisical`
  - `ci(security): pin third-party actions by commit SHA`
  - `ci(deps): enable Renovate digest pinning so SHA pins stay current`
- Do NOT push or open a PR unless the operator instructed it.

## Steps

### Step 1: Narrow the Infisical secret pull

Change `secret-path: /` to the specific path that holds the Renovate token.
**You cannot discover that path from the repo** — it lives in the Infisical
project. Two acceptable outcomes:

- **Preferred**: if the action supports fetching a single secret by name (check
  `Infisical/secrets-action` inputs for something like `secret-name`), use it
  to fetch only `RENOVATE_TOKEN`.
- **Otherwise**: set `secret-path` to a narrower folder (e.g. `/ci` or
  `/renovate`) **and** leave a comment stating the path must contain
  `RENOVATE_TOKEN`.

Because the correct value is operator knowledge, this step is allowed to end
in a **reported question** rather than a guess: if you cannot determine the
right input from the action's documented inputs, make no change to this line,
and report exactly what the operator must decide (with the two options above).
Do not guess a path — a wrong path breaks the nightly Renovate run silently
(the token resolves empty and the next step fails).

Also add `env:` scoping if the action supports exporting to a step output
rather than the job environment; if it only exports to the environment, note
that in a comment.

**Verify**: workflow YAML parses (command from the table). If you changed the
path, state in your report that the next scheduled Renovate run is the real
verification.

### Step 2: Pin third-party actions by SHA

For each third-party reference in the list above, replace the tag with the
full 40-character commit SHA of that release, keeping the human-readable
version in a trailing comment — the standard form:

```yaml
      - uses: docker/build-push-action@<40-char-sha>  # v7.0.0
```

Resolve each SHA with `gh api repos/<owner>/<repo>/git/ref/tags/<tag> --jq .object.sha`
(if the ref is an annotated tag object, dereference it:
`gh api repos/<owner>/<repo>/git/tags/<sha> --jq .object.sha`). If `gh` is
unavailable or unauthenticated, **stop and report** — do not invent SHAs and
do not skip the comment, since the version comment is what makes the pin
reviewable.

Apply consistently across all four workflow files.

**Verify**: `grep -rnE "uses: [^ ]+@v[0-9]" .github/workflows/` → returns only
the first-party actions you deliberately left tag-pinned (or nothing, if you
pinned everything); workflow YAML parses.

### Step 3: Make Renovate keep the pins current

In `renovate.json`, add `"pinDigests": true` at the top level. This activates
the existing `"pin"`/`"digest"` rule, which already sets `automerge: true` for
those update types, so digest refreshes will flow automatically without
review noise.

If the operator would rather **not** have digest churn, the honest alternative
is to delete `"pin"` and `"digest"` from the first `packageRules` entry so the
config stops implying behavior that does not happen. Choose the first option
(pin + automerge) since step 2 creates pins that will otherwise rot; note the
alternative in your report.

**Verify**: Renovate config parses; `grep -n "pinDigests" renovate.json` →
present.

## Test plan

No unit tests — this is CI configuration. Verification is the parse checks
above plus `npm test` staying green. The real verification is the next
scheduled Renovate run and the next release build; say so in your report and
in the `plans/README.md` status note.

## Done criteria

- [ ] Workflow YAML and `renovate.json` both parse
- [ ] `npm test` exits 0
- [ ] `grep -rn "secret-path: /$" .github/workflows/renovate.yml` → no match,
      OR the report explains exactly why it was left and what the operator
      must decide
- [ ] Third-party actions carry 40-char SHAs with version comments
- [ ] `renovate.json` contains `pinDigests` (or the inert rule entries were
      removed, with the choice explained)
- [ ] No files outside the in-scope list modified (`git status`)
- [ ] `plans/README.md` status row updated

## STOP conditions

- `gh` is unavailable/unauthenticated, so SHAs cannot be resolved
  authoritatively — report; do not hand-write SHAs from memory.
- Plan 004 has not landed and you find yourself wanting to edit `ci.yml`'s
  permissions or `release.yml`'s install step — stay out; those are 004's.
- The Infisical action has no documented input for a single secret and no
  safe narrower path is knowable from the repo — leave the line unchanged and
  report (this is an expected, acceptable outcome for step 1).
- Enabling `pinDigests` appears to conflict with an existing Renovate rule you
  do not understand — report the rule rather than reshuffling `packageRules`
  (order matters in that file: the trailing `major → automerge: false` rule
  is what stops major auto-merges, and moving things can silently re-enable
  them).

## Maintenance notes

- After step 2, updating an action means updating a SHA — that is the point,
  and step 3 automates it. If a human ever needs to bump one by hand, keep the
  `# vX.Y.Z` comment truthful.
- `renovate.json`'s `packageRules` is order-dependent: the final
  `{"matchUpdateTypes": ["major"], "automerge": false}` entry is what prevents
  automerged majors (this repo once took an unreviewed TypeScript major that
  way). Any future reordering must preserve that entry's position last.
- Recorded follow-up, not in this plan: a `customManagers.regex` entry so
  Renovate also tracks the pinned versions inside `Jenkinsfile` and
  `scripts/lint-in-container.sh`, which no manager currently sees.
- Reviewer: confirm every SHA's trailing comment matches the tag it was
  resolved from, and that the Renovate job still names a path containing the
  token.
