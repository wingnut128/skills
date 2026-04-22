# GitHub audit — control checks

Read-only verification that a repo matches the `repo-bootstrap` baseline. Every control the skill normally applies has a corresponding check here.

## Invocation shape

```bash
OWNER=...  # e.g. acme-org
REPO=...   # e.g. widget-service
BRANCH=$(gh api "repos/$OWNER/$REPO" --jq '.default_branch')
VIS=$(gh repo view "$OWNER/$REPO" --json visibility -q .visibility | tr '[:upper:]' '[:lower:]')
OWNER_TYPE=$(gh api "users/$OWNER" --jq '.type // "Organization"' 2>/dev/null || echo "User")
```

`$VIS` ∈ `public | private | internal`. `$OWNER_TYPE` ∈ `User | Organization`. These two drive the skip logic below.

**Note on visibility casing:** GitHub's API returns visibility as uppercase (`"PUBLIC"`, `"PRIVATE"`, `"INTERNAL"`) when queried via `gh repo view --json visibility`, but the older `.visibility` field in `gh api repos/{owner}/{repo}` returns lowercase. Normalize to lowercase at the top of the script (as shown above) to avoid subtle skip-logic bugs.

## Status vocabulary

- ✅ **PASS** — control matches baseline
- ❌ **FAIL** — control exists but doesn't match baseline. Show `current → expected`
- ⚠️ **SKIP** — control doesn't apply to this repo (e.g. fork-PR approval on a private repo)
- ❓ **UNKNOWN** — couldn't read state (permissions error, API change, etc.). Don't assume pass

## File-presence controls

Each check is a file existence test. Run from the repo root.

| Control | Check |
|---|---|
| LICENSE exists | `test -f LICENSE \|\| test -f LICENSE.md \|\| test -f LICENSE.txt` |
| SECURITY.md exists | `test -f SECURITY.md \|\| test -f .github/SECURITY.md` |
| CODEOWNERS exists | `test -f CODEOWNERS \|\| test -f .github/CODEOWNERS \|\| test -f docs/CODEOWNERS` |
| CLAUDE.md exists | `test -f CLAUDE.md` |
| .gitignore exists | `test -f .gitignore` |
| Dependabot config | `test -f .github/dependabot.yml \|\| test -f .github/dependabot.yaml` |

Dependabot is only a PASS if the config covers every package ecosystem detected in the repo (Cargo → `package-ecosystem: "cargo"`, npm/bun → `"npm"`, etc.). Simple presence is a partial pass — note gaps.

## Branch-protection controls

Fetch once, query many:

```bash
BP=$(gh api "repos/$OWNER/$REPO/branches/$BRANCH/protection")
```

| Control | jq filter | Expected |
|---|---|---|
| Required status checks present | `.required_status_checks.contexts \| length` | `> 0` |
| Dismiss stale reviews | `.required_pull_request_reviews.dismiss_stale_reviews` | `true` |
| Require code-owner review | `.required_pull_request_reviews.require_code_owner_reviews` | `true` |
| Enforce admins | `.enforce_admins.enabled` | `true` |
| Conversation resolution required | `.required_conversation_resolution.enabled` | `true` |
| Force pushes blocked | `.allow_force_pushes.enabled` | `false` |
| Branch deletion blocked | `.allow_deletions.enabled` | `false` |

If `$BP` fetch returns 404, the branch has **no protection at all** — report that as a single top-level FAIL and skip the per-field checks.

## Repo-level controls

```bash
REPO_INFO=$(gh api "repos/$OWNER/$REPO")
```

| Control | jq filter | Expected |
|---|---|---|
| Auto-merge enabled | `.allow_auto_merge` | `true` |
| Delete branch on merge | `.delete_branch_on_merge` | `true` |
| Secret scanning | `.security_and_analysis.secret_scanning.status` | `enabled` |
| Secret scanning push protection | `.security_and_analysis.secret_scanning_push_protection.status` | `enabled` |

**Secret scanning SKIP condition:** private repo without GHAS. Detect:

```bash
[ "$VIS" != "public" ] && [ "$(jq -r '.security_and_analysis.advanced_security.status // "disabled"' <<< "$REPO_INFO")" != "enabled" ] && echo "SKIP: private repo without GHAS"
```

## Actions permission controls

| Control | API | Expected |
|---|---|---|
| Default workflow token permissions | `actions/permissions/workflow` → `.default_workflow_permissions` | `read` |
| GITHUB_TOKEN can approve PRs | `actions/permissions/workflow` → `.can_approve_pull_request_reviews` | `false` |
| Fork-PR contributor approval | `actions/permissions/fork-pr-contributor-approval` → `.approval_policy` | `all_external_contributors` |
| Reusable-workflow access level | `actions/permissions/access` → `.access_level` | `none` (private user-owned, private org-owned without shared workflows) |

**Fork-PR approval SKIP condition:** private repo. The API rejects with `"Fork PR approval is not allowed for private repositories"`. Check `$VIS`.

**Access-level SKIP condition:** public repo. The API rejects with `"Access policy only applies to internal and private repositories"`. Check `$VIS`.

**Access-level FAIL nuance:** `none` is the recommended floor, but `organization` is acceptable for org-owned repos that intentionally share workflows. Report `organization` as ⚠️ WARN with a note rather than FAIL when `$OWNER_TYPE == "Organization"`.

## Vulnerability intake controls

| Control | API | Expected |
|---|---|---|
| Dependabot alerts | `repos/{owner}/{repo}/vulnerability-alerts` | HTTP 204 (enabled) |
| Dependabot security updates | `repos/{owner}/{repo}/automated-security-fixes` → `.enabled` | `true` (HTTP 200 with JSON body `{enabled, paused}`) |
| Private vulnerability reporting | `repos/{owner}/{repo}/private-vulnerability-reporting` → `.enabled` | `true` |

**`vulnerability-alerts`** is a 204-vs-404 endpoint — no body. Use `gh api -i` and grep for `HTTP/2.0 204` (or `HTTP/1.1 204`). 404 means disabled.

**`automated-security-fixes`** returns HTTP 200 with a JSON body `{"enabled": bool, "paused": bool}`. A PASS requires `.enabled == true`; flag `.paused == true` as a ⚠️ WARN since security-update PRs have been temporarily silenced even though the setting is on.

These two endpoints are shaped differently despite being "the same family" — the 204 pattern does not apply to `automated-security-fixes`. Don't use a shared check for both.

## Output format

Print a scoreboard with fixed-width columns. Group by category. Close with a summary line: `N pass / M fail / K skip`.

```text
=== acme-org/widget-service — repo-bootstrap audit ===
Visibility: public   Owner: User   Default branch: main

[Files]
✅ LICENSE exists
✅ SECURITY.md exists
✅ CODEOWNERS exists
✅ CLAUDE.md exists
✅ .gitignore exists
✅ Dependabot config (cargo ecosystem covered)

[Branch protection — main]
✅ Required status checks (5 contexts)
✅ Dismiss stale reviews
✅ Require code-owner review
✅ Enforce admins
✅ Conversation resolution required
✅ Force pushes blocked
✅ Branch deletion blocked

[Repo settings]
✅ Auto-merge enabled
✅ Delete branch on merge
✅ Secret scanning (public-free)
✅ Secret scanning push protection

[Actions permissions]
✅ Default workflow token: read
✅ GITHUB_TOKEN cannot approve PRs
✅ Fork-PR contributor approval: all_external_contributors
⚠️ SKIP Reusable-workflow access level (public repo — N/A)

[Vulnerability intake]
✅ Dependabot alerts enabled
✅ Dependabot security updates enabled
✅ Private vulnerability reporting enabled

─────────────────────────────────────────
Summary: 19 pass / 0 fail / 1 skip
```

## After the report

If any controls FAILed, offer to re-enter normal bootstrap mode and apply just the failing items. Phrase it as a question — audit mode is read-only by contract, switching to write mode is a user choice.
