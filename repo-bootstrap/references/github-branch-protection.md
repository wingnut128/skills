# GitHub branch protection — API payloads

All calls go through `gh api` with an authenticated `gh` CLI session.

## 1. Branch protection

Endpoint: `PUT /repos/{owner}/{repo}/branches/{branch}/protection`

```json
{
  "required_status_checks": {
    "strict": false,
    "contexts": [
      "Analyze (javascript-typescript)",
      "Scan",
      "semgrep-cloud-platform/scan",
      "security/snyk (owner)"
    ]
  },
  "enforce_admins": true,
  "required_pull_request_reviews": {
    "dismiss_stale_reviews": true,
    "require_code_owner_reviews": true,
    "require_last_push_approval": false,
    "required_approving_review_count": 0
  },
  "restrictions": null,
  "required_conversation_resolution": true,
  "allow_force_pushes": false,
  "allow_deletions": false
}
```

**Notes:**
- `strict: false` means PRs don't have to be up-to-date with `main` before merging. Set to `true` only if you have few PRs at a time — otherwise stacked PRs become painful.
- Context names come from the `name:` field of each workflow job (not the workflow's top-level `name:`). GitHub Actions jobs surface as `<job name>` or `<job name> (<matrix-value>)` when matrixed.
- `require_code_owner_reviews: true` + `required_approving_review_count: 0` means code-owner review is required but a reviewer need not explicitly "approve" — PR authors can self-merge if they are the only code owner (useful for solo projects). Do **not** raise the count to 1 on solo repos: GitHub blocks self-approval, so a sole code owner ends up unable to merge their own PRs without admin override.
- `restrictions: null` means no user/team restriction on pushing to the branch. Set to `{"users": [], "teams": [...]}` to restrict.
- `required_conversation_resolution: true` forces every review thread to be resolved before merge — cheap guardrail against losing track of feedback.
- `allow_force_pushes: false` and `allow_deletions: false` are the defaults, but set them explicitly so re-runs of the skill don't drift if the defaults ever change.

Invoke:

```bash
gh api -X PUT "repos/$OWNER/$REPO/branches/$BRANCH/protection" \
  -H "Accept: application/vnd.github+json" \
  --input payload.json
```

## 2. Fork PR workflow approval

**Public repos only.** On private repos this API returns HTTP 422 with "Fork PR approval is not allowed for private repositories" — private repos can't be forked by outside collaborators, so the setting is moot. Check `gh repo view <owner>/<repo> --json isPrivate -q .isPrivate` before calling.

Endpoint: `PUT /repos/{owner}/{repo}/actions/permissions/fork-pr-contributor-approval`

```json
{ "approval_policy": "all_external_contributors" }
```

Values:
- `all_external_contributors` — strictest. Any PR from someone without write access requires manual approval before workflows run.
- `first_time_contributors_new_to_github` — loosest of the gated options. Only gates brand-new GitHub accounts.
- `first_time_contributors` — middle option.

The strictest setting protects repo secrets (e.g., Snyk token, deploy keys) from being exfiltrated by a hostile fork PR.

## 3. Workflow permissions

### 3a. Default workflow token permissions

Endpoint: `PUT /repos/{owner}/{repo}/actions/permissions/workflow`

```json
{
  "default_workflow_permissions": "read",
  "can_approve_pull_request_reviews": false
}
```

Read-only default means workflows can't mutate repo state (issues, comments, releases) unless a job explicitly opts in with its own `permissions:` block. This is the OpenSSF recommendation.

### 3b. Reusable-workflow access level

Endpoint: `PUT /repos/{owner}/{repo}/actions/permissions/access`

```json
{ "access_level": "none" }
```

Controls which **other** repos can call reusable workflows or use composite actions defined in *this* repo.

**Scope:** only applies to **private** and **internal** repositories. Calling this endpoint on a public repo returns HTTP 422 with `"Access policy only applies to internal and private repositories."` — skip entirely for public repos. Public repo workflows are inherently callable from anywhere; there's no access-level concept.

Detect first:
```bash
VIS=$(gh repo view "$OWNER/$REPO" --json visibility -q .visibility)  # "public" | "private" | "internal"
[ "$VIS" = "public" ] && echo "skip — public repo has no access policy"
```

Values (for private/internal repos):
- `none` — only this repo can use its own workflows. Tightest. Right default for private **user-owned** repos and private org repos that don't intentionally share workflows.
- `organization` — any repo in the same org can consume. GitHub's platform default for private org-owned repos. Fine when internal sharing is expected.
- `enterprise` — enterprise-wide sharing. Only on GHEC with an enterprise account.

Pick:
- Private user-owned → `none`
- Private org-owned with no shared workflows → `none`
- Private org-owned that hosts shared workflows → `organization` (the default; set explicitly only to pin it against drift)

## 4. Repo-level merge settings

Endpoint: `PATCH /repos/{owner}/{repo}`

```bash
gh api -X PATCH "repos/$OWNER/$REPO" \
  -F allow_auto_merge=true \
  -F delete_branch_on_merge=true
```

- `allow_auto_merge: true` enables the "Enable auto-merge" button on PRs. It's safe even on solo repos: auto-merge still respects required status checks and the code-owner review rule — it only fires once the branch-protection gate opens, and admin bypass does **not** apply to auto-merge.
- `delete_branch_on_merge: true` removes merged topic branches automatically. Pairs well with Dependabot and short-lived feature branches.

Leave `allow_merge_commit`, `allow_squash_merge`, `allow_rebase_merge` alone unless the user has a preference — defaults are all three enabled.

## 5. Security & scanning toggles

### 5a. Dependabot alerts + security updates

Simple enable toggles (no body):

```bash
gh api -X PUT "repos/$OWNER/$REPO/vulnerability-alerts"
gh api -X PUT "repos/$OWNER/$REPO/automated-security-fixes"
```

`vulnerability-alerts` enables Dependabot scanning; `automated-security-fixes` enables Dependabot-opened PRs for vulnerable dependencies.

### 5b. Secret scanning + push protection

Endpoint: `PATCH /repos/{owner}/{repo}`

```bash
gh api -X PATCH "repos/$OWNER/$REPO" \
  -F "security_and_analysis[secret_scanning][status]=enabled" \
  -F "security_and_analysis[secret_scanning_push_protection][status]=enabled"
```

- **Secret scanning** detects committed secrets (API keys, tokens) post-push and opens alerts.
- **Push protection** rejects the push itself when a high-confidence secret pattern matches — prevents the leak entirely.

**Availability:**
- Public repos: **free**, always available.
- Private repos: require **GitHub Advanced Security** (paid). On a private repo without GHAS, the API returns HTTP 422 with `"Secret scanning is not available for this repository"`. Catch and skip with a note, don't treat as fatal.

Check before calling to avoid the 422:
```bash
IS_PRIVATE=$(gh repo view "$OWNER/$REPO" --json isPrivate -q .isPrivate)
HAS_GHAS=$(gh api "repos/$OWNER/$REPO" --jq '.security_and_analysis.advanced_security.status // "disabled"')
if [ "$IS_PRIVATE" = "true" ] && [ "$HAS_GHAS" != "enabled" ]; then
  echo "Skipping secret scanning: private repo without GHAS"
else
  # apply the PATCH
fi
```

### 5c. Private vulnerability reporting

Endpoint: `PUT /repos/{owner}/{repo}/private-vulnerability-reporting`

```bash
gh api -X PUT "repos/$OWNER/$REPO/private-vulnerability-reporting"
```

No body. Enables the "Report a vulnerability" button in the Security tab, letting outside researchers file **private** advisories instead of opening public issues. Available on all public repos at no cost.

Pairs well with `SECURITY.md` — the policy tells researchers how to report, this endpoint gives them the channel.

## Discovering status check contexts

Before setting required checks, discover which jobs a workflow produces. For each `.github/workflows/*.yml`:

- The workflow file has a top-level `name:` (workflow name, appears once) and job-level `name:` inside each `jobs.<id>:` entry.
- The **status check context** is the job `name:` as GitHub displays it.
- If a job uses `strategy.matrix`, the context includes the matrix values: `<job name> (<matrix values joined by ", ">)`.

Example — `codeql.yml`:

```yaml
jobs:
  analyze:
    name: Analyze
    strategy:
      matrix:
        language: [javascript-typescript]
```

Produces one status check context: `Analyze (javascript-typescript)`.

If uncertain, run a PR through and read the resulting check contexts from `gh pr checks <PR>` output.
