---
name: repo-bootstrap
description: >-
  Use when starting a new GitHub or Forgejo repo, hardening an existing one
  that lacks baseline security controls, or auditing a repo's current posture.
  Invoke for requests like "set up a new repo", "add branch protection",
  "bootstrap this repo", "harden this repo", "add security policy",
  "add license", "set up code owners", "configure dependabot",
  "audit this repo", "check my repo posture", or any ask that touches
  repo-level security and governance. Applies the full baseline — license,
  security policy, CODEOWNERS, security-hardened .gitignore, dependency
  update config, branch protection with required status checks, fork-PR
  workflow approval gating, secret scanning, private vulnerability reporting,
  and Dependabot/security alerts. Supports a read-only audit sub-mode that
  scores an existing repo against the baseline. Idempotent — detects what's
  already present and only adds what's missing, so it is safe to re-run.
---

# repo-bootstrap

Apply a consistent baseline of security controls and governance files to a fresh or under-protected repo. Works on both GitHub and Forgejo (Gitea-compatible API). Idempotent — safe to re-run.

## When this skill fires

Trigger on requests like:

- "Set up a new repo"
- "Bootstrap this repo"
- "Add branch protection"
- "Harden this repo"
- "Add a license / security policy / CODEOWNERS"
- "Configure dependabot / renovate"
- "Require status checks before merge"
- "Audit this repo" / "check my repo posture" / "is this repo hardened" → runs **audit mode** (read-only scoreboard; see § Audit mode)

Do not fire for narrow asks that clearly fit a single existing tool, like "add one rule to .gitignore" or "write a commit message" — those are one-file edits, not repo bootstrapping.

## Principles

- **Detect, don't assume.** Read the current state (remote URL, `.github/workflows/`, existing files, branch protection) before doing anything. Skip anything already in place.
- **Confirm before applying branch protection.** It's not hard to revert, but it's visible to the user and affects future workflow, so surface the full plan first.
- **Never overwrite a non-empty file without explicit permission.** If `LICENSE` already exists with content, leave it; note that it's already there.
- **Prefer fewer, tighter questions.** The user probably knows what they want. Ask once per decision, don't belabor.
- **Explain what's being skipped on Forgejo.** A few controls (GitHub private security advisories, `fork-pr-contributor-approval`) don't have exact Forgejo equivalents. Tell the user what's being skipped so they can apply an alternative.

## Overall flow

1. **Detect context** (no user input needed): forge type, repo owner/name, branches, existing files, existing workflows, current protection.
2. **Present the plan** to the user: what will be created, what will be configured, what's already in place and will be skipped.
3. **Prompt for choices** not inferrable from the repo: license type, security contact style, primary stack (for `.gitignore` + dependency updates), required review count.
4. **Apply** in this order:
   1. Write baseline files (LICENSE, SECURITY.md, CODEOWNERS, .gitignore, dependency-update config, CLAUDE.md stub if absent)
   2. Evaluate release automation — if the repo merits one (see step 4b skip criteria), add release-plz/release-please workflows + config
   3. Commit those to a feature branch and open a PR (or commit directly to main if user prefers — ask)
   4. After that PR is merged, or in parallel on another PR, apply branch protection and account-level toggles via the forge API
5. **Verify** by reading back the current state and reporting anything that didn't stick.

## Step 1 — Detect context

Run in parallel:

```bash
# Forge + repo identity
git remote get-url origin
git rev-parse --abbrev-ref HEAD

# Existing workflow files (for required-check context discovery)
ls .github/workflows/ 2>/dev/null
ls .forgejo/workflows/ 2>/dev/null

# What baseline files are already present
ls -la LICENSE* SECURITY.md CODEOWNERS .github/CODEOWNERS .gitignore .github/dependabot.yml renovate.json CLAUDE.md 2>/dev/null
```

Parse the remote URL:

- Host `github.com` → forge = `github`, use `gh` CLI for API calls
- Any other host → forge = `forgejo` (or Gitea — same API), use `curl` against `<host>/api/v1/` with a `FORGEJO_TOKEN` env var

If the remote isn't `origin`, ask the user for the canonical remote name.

## Step 2 — Present the plan

Tell the user, in plain language, what you're about to do. Something like:

> I'll set up repo hardening on `owner/name` (detected: GitHub, main branch).
> I'll create these files (none exist yet): LICENSE, SECURITY.md, CODEOWNERS, .gitignore additions, dependabot.yml.
> CLAUDE.md already exists — skipping.
> For branch protection I'll require these checks (found in `.github/workflows/`): CodeQL, Semgrep.
> I'll also enable fork-PR approval gating and set default workflow permissions to read-only.
> Sound good?

Wait for confirmation before applying anything API-side. File writes can go into a PR the user can review, but branch-protection changes are immediate.

## Step 3 — Prompt for choices

Ask only for what can't be inferred. Defaults in parens:

- **License** (MIT): MIT / Apache-2.0 / BSD-3-Clause / proprietary
- **Security reporting path**:
  - On GitHub: private advisory (default) or email
  - On Forgejo: email (default — Forgejo doesn't have GitHub-style advisories)
- **Primary stack** (infer from files — `package.json` → bun/node, `Cargo.toml` → rust, `pyproject.toml`/`requirements.txt` → python, `go.mod` → go; ask only if ambiguous)
- **Required approval count** (0): 0 for solo repos, 1+ for teams

If the user is clearly working solo on a personal repo, don't belabor the approval question — 0 is fine.

## Step 4 — Apply

### 4a. Baseline files

Pull each file from `templates/` in this skill, substituting placeholders:

- `{{YEAR}}` → current year
- `{{OWNER}}` → copyright holder (from git config user.name)
- `{{REPO}}` → repo name
- `{{FORGE_URL}}` → base URL of the forge
- `{{SECURITY_REPORT_URL}}` → the advisory link or `mailto:...`
- `{{CLAUDE_USERNAME}}` → the user's forge username, for CODEOWNERS

Files to write (only if absent):

| File | Template | Notes |
|---|---|---|
| `LICENSE` | `templates/LICENSE-<choice>.txt` | |
| `SECURITY.md` | `templates/SECURITY-github.md` or `SECURITY-forgejo.md` | |
| `.github/CODEOWNERS` (GH) or `.gitea/CODEOWNERS` (Forgejo) | `templates/CODEOWNERS` | |
| `.gitignore` | Combine `templates/gitignore-security-base` + `templates/gitignore-<stack>` | **Always append the security base**, even if `.gitignore` already exists — merge intelligently, don't duplicate entries the user already has |
| `.github/dependabot.yml` (GH) or `renovate.json` (Forgejo) | `templates/dependabot-<stack>.yml` / `templates/renovate-<stack>.json` | |
| `CLAUDE.md` | `templates/CLAUDE.md.stub` | Only if absent; this is a stub, not a full template |

For `.gitignore` specifically: read the existing file if any, parse it into a set of lines, union with the template lines, preserve the existing section comments, and write the union back. The goal is that re-running the skill on a repo that already has the security block doesn't create duplicate lines.

Commit these to a feature branch and open a PR. The branch name convention: `repo-bootstrap/baseline`.

### 4b. Release automation (optional — evaluate first)

Set up **release-plz** (Rust) or **release-please** (Node/Python/Go) to replace hand-rolled version-bump + CHANGELOG flows. The tool keeps an open Release PR that auto-bumps version and regenerates CHANGELOG from conventional commits; merging the PR creates a tag + draft GitHub Release. A separate tag-triggered workflow builds binaries, uploads them, and un-drafts the release.

**Skip this step entirely when any of the following are true:**

- No versioned distribution — internal service, CI-deployed app, personal scratch tool
- Fork of an upstream project that owns its own releases
- Existing working release tool already wired (goreleaser, changesets, cargo-release, semantic-release) — don't replace it unless the user asks
- Pre-0.1.0 with high churn and no external consumers
- Monorepo with bespoke per-package release choreography
- User explicitly declines when asked

**Ask once, with a default:**

> I noticed this repo has no automated release flow. I can add one based on release-plz — on every push to main it opens a "Release PR" that auto-bumps \`Cargo.toml\` and regenerates \`CHANGELOG.md\` from conventional commits. Merging it tags + releases. Worth setting up? (Y/n)

If yes, apply:

**Rust toolchain note.** The Rust templates use inline `rustup update stable && rustup default stable` instead of `dtolnay/rust-toolchain@stable`. The action's only ref is a rolling branch, which — once SHA-pinned for pin-check compliance — freezes the toolchain and can't be dependabot-updated. `rustup` on the GitHub-hosted runner is pre-installed and picks up the latest stable on every run, with no `uses:` line to pin.

1. **Pin action SHAs first.** Look up current SHAs for:
   - `step-security/harden-runner` (latest v2.x)
   - `actions/checkout` (latest v6+)
   - `actions/cache` (latest v5+)
   - `actions/upload-artifact` (latest v7+)
   - `actions/download-artifact` (latest v8+)
   - `release-plz/action` (latest v0.5.x) — resolve annotated tag to commit SHA:

     ```bash
     # 1) Get annotated-tag object SHA
     TAG_OBJ=$(gh api repos/release-plz/action/git/refs/tags/v0.5.128 --jq '.object.sha')
     # 2) Resolve to commit SHA
     gh api repos/release-plz/action/git/tags/$TAG_OBJ --jq '.object.sha'
     ```

2. **Write templates**, substituting placeholders:

   | File | Template | Placeholders |
   |---|---|---|
   | `release-plz.toml` | `templates/release-plz.toml` | (none) |
   | `.github/workflows/release-plz.yml` | `templates/release-plz.yml` | `{{DEFAULT_BRANCH}}`, `{{HARDEN_RUNNER_SHA/VERSION}}`, `{{CHECKOUT_SHA/VERSION}}`, `{{RELEASE_PLZ_SHA/VERSION}}` |
   | `.github/workflows/release.yml` | `templates/release-tag-build.yml` | Above plus `{{CRATE_NAME}}`, `{{CACHE_SHA/VERSION}}`, `{{UPLOAD_ARTIFACT_SHA/VERSION}}`, `{{DOWNLOAD_ARTIFACT_SHA/VERSION}}` |

3. **If a `release.yml` already exists**, don't overwrite — show the user what the new tag-triggered flow looks like and ask whether to replace. The existing one probably uses Cargo.toml-diff detection, which conflicts with release-plz.

4. **Tell the user to create a PAT**:
   > Before merging, create a **fine-grained PAT** (github.com → Settings → Developer settings → Personal access tokens → Fine-grained) scoped to this repo with **Contents: Read and write** and **Pull requests: Read and write**. Store it as repo secret `RELEASE_PLZ_TOKEN`. `GITHUB_TOKEN` can't trigger CI on the Release PR, which is why the PAT is needed.

5. **Verify on first push to main**: the `Release-plz` workflow should run and open a PR targeting the next version. If it doesn't appear within ~1 minute, check `gh run list --workflow=release-plz.yml` for errors (most common cause: PAT missing or missing permissions).

**Language mapping (pick one based on stack detected in step 3):**

| Stack | Tool | Template exists in this skill |
|---|---|---|
| Rust binary | release-plz | ✅ |
| Rust library (crates.io) | release-plz with `publish = true` + `CARGO_REGISTRY_TOKEN` | Adapt from Rust binary template |
| Node / Python / Go | release-please (`googleapis/release-please-action`) | Not yet — adapt the Rust workflow structure, swap the action and config file |
| Go binary | goreleaser if user prefers, otherwise release-please + tag-triggered `go build` matrix | Not yet |

If no template exists yet for the user's stack, write the workflow by analogy to `release-plz.yml` (same structure: release-PR job + release job + separate tag-triggered build job). Don't bluff — tell the user this is the first time you're wiring their stack and to review carefully.

### 4c. Branch protection + account toggles

For the details of the API calls, see:

- `references/github-branch-protection.md`
- `references/github-audit.md` (read-only audit control list — used by § Audit mode)
- `references/github-rulesets.md` (upgrade path — see below)
- `references/forgejo-branch-protection.md`

**Rulesets vs classic protection.** The default path above uses **classic branch protection** (`PUT /branches/{branch}/protection`). If the user asks about bypass allowances (e.g. "let me bypass the review count but nobody else"), targeting multiple branches with one rule (`main`, `release/*`), required signed commits on a branch, push-time rulesets ("block direct pushes to main"), or commit-message patterns — see `references/github-rulesets.md`. Rulesets and classic protection coexist (their rules are unioned), so adding a ruleset doesn't require removing classic. Full classic→rulesets migration is out of scope for this skill; see Stage E in `TODO.md`.

High-level:

**GitHub:**

1. Discover which workflow jobs produce status check contexts (parse `.github/workflows/*.yml`; each job's `name:` field maps to a context). If no workflows exist yet, pass `required_status_checks: null` — don't send an empty contexts array.
2. `PUT /repos/{owner}/{repo}/branches/main/protection` with:
   - `required_status_checks`: `strict: false`, contexts from step 1 plus any the user adds (or `null` if no workflows)
   - `required_pull_request_reviews`: dismiss stale, require code owners, N approvals
   - `enforce_admins`: true
   - `restrictions`: null
3. **Only if the repo is public**: `PUT /repos/{owner}/{repo}/actions/permissions/fork-pr-contributor-approval` with `approval_policy: all_external_contributors`. This API rejects the call on private repos with "Fork PR approval is not allowed for private repositories" — private repos can't be forked by outside collaborators, so the setting is moot. Detect private vs public via `gh repo view --json isPrivate` before calling.
4. `PUT /repos/{owner}/{repo}/actions/permissions/workflow` with `default_workflow_permissions: read`
5. **Only if the repo is private or internal**: `PUT /repos/{owner}/{repo}/actions/permissions/access` with `access_level: none` (private user-owned, or private org-owned without shared workflows) or `organization` (private org-owned with intentionally shared workflows). Restricts which *other* repos can consume this repo's reusable workflows. This API returns HTTP 422 on public repos — detect via `gh repo view --json visibility` and skip when `visibility == "public"`.
6. `PUT /repos/{owner}/{repo}/vulnerability-alerts` (enables Dependabot alerts)
7. `PUT /repos/{owner}/{repo}/automated-security-fixes` (enables Dependabot security updates)
8. **Only if public OR private-with-GHAS**: `PATCH /repos/{owner}/{repo}` with `security_and_analysis.secret_scanning.status: enabled` and `secret_scanning_push_protection.status: enabled`. On private repos without GitHub Advanced Security, this returns HTTP 422 — detect and skip with a note. Check via `gh api repos/{owner}/{repo} --jq '.security_and_analysis.advanced_security.status'`.
9. `PUT /repos/{owner}/{repo}/private-vulnerability-reporting` (enables private advisory intake — pairs with `SECURITY.md`)
10. `PATCH /repos/{owner}/{repo}` with `allow_auto_merge: true` and `delete_branch_on_merge: true`

**Forgejo:**

1. Discover which workflow jobs produce required checks (parse `.forgejo/workflows/*.yml` OR `.github/workflows/*.yml` if Forgejo Actions is configured to read from there).
2. `POST /api/v1/repos/{owner}/{repo}/branch_protections` (or PATCH if one already exists) with:
   - `rule_name: main`
   - `enable_push: false`
   - `enable_merge_whitelist: false` (unless user wants to restrict)
   - `required_approvals: N`
   - `block_on_rejected_reviews: true`
   - `block_on_outdated_branch: false`
   - `enable_status_check: true`
   - `status_check_contexts: [...]`
   - `enable_approvals_whitelist: false`
   - `require_signed_commits: false` (unless user asks)

   See `references/forgejo-branch-protection.md` for the full payload.

Things to skip on Forgejo and tell the user:

- No `fork-pr-contributor-approval` equivalent — Forgejo Actions secrets don't have a gating policy in the same shape.
- No private security advisories — `SECURITY-forgejo.md` points to an email or the repo's issue tracker.
- Dependabot alerts don't exist on Forgejo. Renovate running as a scheduled workflow is the nearest equivalent; the skill writes `renovate.json` but **does not** install a runner — the user has to wire up Renovate themselves or install the Forgejo bot.

## Step 5 — Verify

After the API calls, read back the state:

**GitHub:**

```bash
gh api repos/{owner}/{repo}/branches/main/protection | jq '{required_status_checks, required_pull_request_reviews, enforce_admins, required_conversation_resolution, allow_force_pushes, allow_deletions}'
gh api repos/{owner}/{repo}/actions/permissions/fork-pr-contributor-approval
gh api repos/{owner}/{repo}/actions/permissions/workflow
# actions/permissions/access only applies to private/internal repos; returns 422 on public
[ "$(gh repo view {owner}/{repo} --json visibility -q .visibility)" != "public" ] && gh api repos/{owner}/{repo}/actions/permissions/access
gh api repos/{owner}/{repo} | jq '{allow_auto_merge, delete_branch_on_merge, secret_scanning: .security_and_analysis.secret_scanning.status, secret_scanning_push_protection: .security_and_analysis.secret_scanning_push_protection.status}'
gh api repos/{owner}/{repo}/private-vulnerability-reporting
```

**Forgejo:**

```bash
curl -H "Authorization: token $FORGEJO_TOKEN" "$FORGE_URL/api/v1/repos/$OWNER/$REPO/branch_protections"
```

Report what's now in place vs what was skipped, in a short summary the user can eyeball.

## Audit mode

Triggered when:

- The user explicitly asks ("audit this repo", "check my repo posture", "what's missing", "is this repo hardened")
- The user invokes the skill against a repo that already has most baseline files in place (LICENSE, SECURITY.md, CODEOWNERS, .gitignore, Dependabot config, branch protection on default branch). In that case, offer audit mode before applying anything: "This repo already has most of the baseline. Want me to run an audit instead?"

**Audit is read-only.** No `PUT`/`PATCH`/`POST` calls. Only `GET` queries against the forge API plus local file-existence tests.

**What it does:**

1. Detect context (same as Step 1 — forge, owner, repo, default branch, visibility).
2. Run every check in `references/github-audit.md` (GitHub) or `references/forgejo-branch-protection.md`'s verification section (Forgejo — currently less comprehensive).
3. Print a grouped scoreboard: `[Files]`, `[Branch protection]`, `[Repo settings]`, `[Actions permissions]`, `[Vulnerability intake]`.
4. Close with a summary line: `N pass / M fail / K skip`.

**Status symbols:**

- ✅ PASS
- ❌ FAIL (shows `current → expected`)
- ⚠️ SKIP (control doesn't apply to this repo — e.g., fork-PR approval on a private repo, secret scanning on private-without-GHAS, access-level on public repo)
- ❓ UNKNOWN (couldn't read state — never treat as pass)

**After the report:**

- If any FAILs, ask the user: "Want me to apply fixes for the N failing items?" If yes, re-enter normal apply mode (Step 4) but scoped only to the failing controls.
- If all PASS or SKIP, just confirm the repo is at baseline and stop.

See `references/github-audit.md` for the full control list, `jq` filters, and output format.

## Dry-run mode

If the user says "plan" or "dry run" or passes `--plan`, do everything in steps 1–3 plus generate the file contents and API payloads, but **don't write anything and don't call the forge API**. Print the intended diff and payloads instead, then wait for the user to confirm before actually applying.

## Important edge cases

- **Non-`main` default branch**: use `git symbolic-ref refs/remotes/origin/HEAD` to resolve the default; don't hardcode `main`.
- **Fresh repo with no commits**: some API calls fail without a commit on the default branch. If detected, make the first commit (the baseline files) and push before calling branch-protection APIs.
- **User already has CODEOWNERS**: don't overwrite. Offer to add their username if missing.
- **User has a root `.github/` directory but CODEOWNERS is in repo root**: both locations work on GitHub. Prefer `.github/CODEOWNERS` if neither exists; otherwise leave the existing one alone.
- **Token missing on Forgejo**: prompt the user for a `FORGEJO_TOKEN` with repo admin scope. Don't try to proceed without it.

## CI hardening

Supply-chain and image-level hardening templates. These are **opt-in**: apply when the repo ships a container image or wants defense-in-depth against action-tag attacks. They layer on top of the baseline from Step 4 — don't treat them as required for every repo.

### Runner hardening (StepSecurity `harden-runner`)

Every workflow template in this skill uses `step-security/harden-runner` as its first step. It's a free GitHub Action that monitors (and optionally blocks) outbound network calls from the runner, detecting credential exfiltration or malicious action behavior. The `aquasecurity/trivy-action` compromise (March 2026) is the canonical example of what this defends against — harden-runner would have flagged the infected release binary's outbound callbacks.

**For repos with existing workflows not covered by this skill's templates**, add it as the first step:

```yaml
      - name: Harden the runner
        uses: step-security/harden-runner@<current-SHA> # v2.x
        with:
          egress-policy: audit
```

Start with `audit` mode (logs only). After a couple of runs, review the egress summary at stepsecurity.io and switch to `block` with an allowlist if the egress pattern is stable. Free tier is sufficient for solo/small-team repos; paid tier adds historical dashboards and centralized policy.

Not a hard requirement — skip when the repo only runs trusted first-party code with no third-party actions, or the user explicitly opts out.

### Image + workflow-pinning templates

**When to apply each piece:**

| Template | Apply when | Skip when |
|---|---|---|
| `templates/Dockerfile.chainguard` | Repo builds + distributes a container image | No container distribution; or existing Dockerfile is already distroless and digest-pinned |
| `templates/workflows/image-scan.yml` | Dockerfile is present (upstream or template-applied) | No Dockerfile; or user already runs a paid scanner (Snyk, Prisma) they prefer |
| `templates/workflows/pin-check.yml` | Any repo with `.github/workflows/**` | Repo has no Actions workflows |

### Dockerfile.chainguard

Chainguard `static` distroless runtime + Alpine Rust builder (Chainguard's `rust:latest-dev` lacks a musl rust-std target, so Alpine stays for the build stage — the runtime is what matters for CVE surface).

To apply:

1. Copy `templates/Dockerfile.chainguard` to the repo root as `Dockerfile`.
2. Replace `<BINARY_NAME>` with the crate's binary name.
3. Resolve `<RUNTIME_DIGEST>` from `cgr.dev/chainguard/static:latest`:

   ```bash
   docker pull cgr.dev/chainguard/static:latest
   docker inspect cgr.dev/chainguard/static:latest --format '{{index .RepoDigests 0}}'
   ```

4. Ensure `.github/dependabot.yml` includes a `package-ecosystem: docker` entry so Dependabot keeps base-image digests current.
5. If the binary needs glibc, swap the runtime image to `cgr.dev/chainguard/glibc-dynamic`; if it needs a shell at runtime, swap to `cgr.dev/chainguard/wolfi-base`. The distroless default has no shell, so any HEALTHCHECK must be delegated to the orchestrator (k8s probe, compose healthcheck).
6. Remove or replace the "Optional asset build stage" comment block if the project bundles non-Rust assets (SPA, templates); wire the output directory with `COPY --from=<stage>`.

### image-scan.yml

Grype vulnerability scan + Syft SBOM (CycloneDX + SPDX) + Sigstore SBOM attestation on release. Runs on PRs touching the Dockerfile, weekly cron (drift tracking), and on every published release.

To apply:

1. Copy `templates/workflows/image-scan.yml` → `.github/workflows/image-scan.yml`.
2. Replace `<IMAGE_NAME>` with the image tag base (e.g., `myapp`). It's just a string used at `docker build -t <IMAGE_NAME>:scan .` time — no registry push.
3. Create the drift label used by the weekly cron:

   ```bash
   gh label create image-scan-drift --color c5def5
   ```

4. If the repo isn't Rust, remove `Cargo.lock` from the `paths:` trigger filter and add the lockfile(s) for your stack.

On a `release` event the workflow also attests both SBOMs via `actions/attest-sbom` (Sigstore cosign), attaches them to the GitHub release, and provides release binary provenance when paired with a tag-triggered build workflow.

### pin-check.yml

Runs `pinact` on every PR that touches `.github/workflows/**` or `.github/actions/**` and fails if any `uses:` line is tag-pinned instead of SHA-pinned. Defense-in-depth against action-tag force-push attacks (e.g. the `aquasecurity/trivy-action` incident, March 2026).

To apply:

1. Copy `templates/workflows/pin-check.yml` → `.github/workflows/pin-check.yml`. No substitutions.
2. Add `Verify all actions pinned by SHA` to the required status checks on `main` (see § 4c Branch protection). The context name is the job's `name:` field, not the workflow name.
3. Before the first merge, run `pinact run` locally to SHA-pin any tag-pinned actions already in the repo, otherwise the check will fail on its own introduction PR.

**Gotcha — required checks + `paths:` filters.** If pin-check (or any other required check) uses a `paths:` filter on `pull_request`, GitHub leaves PRs that don't match the filter in a permanent "expected — waiting for status" state and they cannot be merged. The template removes the `paths:` filter from pin-check for exactly this reason. If you add a `paths:` filter back for CI-cost reasons, do not mark the check required — or switch to a branch ruleset with "only when reported" semantics (see `references/github-rulesets.md`).

### Redundant workflows to drop

If the repo has `dependency-review.yml` running on PRs and also has Dependabot + Grype image scanning, the dependency-review action is largely redundant for Rust/compiled-binary projects — it reads `Cargo.lock` diffs and re-scans what Dependabot already alerts on. Safe to remove unless the user relies on its PR-comment UX.

## Resource layout

```text
repo-bootstrap/
├── SKILL.md                              (this file)
├── templates/
│   ├── LICENSE-MIT.txt
│   ├── LICENSE-Apache-2.0.txt
│   ├── LICENSE-BSD-3-Clause.txt
│   ├── SECURITY-github.md
│   ├── SECURITY-forgejo.md
│   ├── CODEOWNERS
│   ├── CLAUDE.md.stub
│   ├── gitignore-security-base
│   ├── gitignore-bun
│   ├── gitignore-node
│   ├── gitignore-rust
│   ├── gitignore-python
│   ├── gitignore-go
│   ├── dependabot-bun.yml
│   ├── dependabot-node.yml
│   ├── dependabot-rust.yml
│   ├── dependabot-python.yml
│   ├── dependabot-go.yml
│   ├── renovate-bun.json
│   ├── renovate-node.json
│   ├── renovate-rust.json
│   ├── renovate-python.json
│   ├── renovate-go.json
│   ├── release-plz.toml                 (Rust release-plz config)
│   ├── release-plz.yml                  (Rust release-plz workflow)
│   ├── release-tag-build.yml            (tag-triggered binary build + release upload)
│   ├── Dockerfile.chainguard            (distroless Rust container template)
│   └── workflows/
│       ├── image-scan.yml               (Grype + Syft SBOM + Sigstore attestation)
│       └── pin-check.yml                (pinact SHA enforcement)
└── references/
    ├── github-branch-protection.md       (payloads + examples)
    ├── github-audit.md                   (read-only control checks + jq filters)
    ├── github-rulesets.md                (upgrade path from classic protection)
    ├── forgejo-branch-protection.md      (payloads + examples)
    └── stack-matrix.md                   (which files go with which stack)
```

Load references on demand — don't read them up front. They're reference material, not part of the main flow.
