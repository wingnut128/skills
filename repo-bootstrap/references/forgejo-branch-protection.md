# Forgejo branch protection — API payloads

Forgejo (and Gitea) expose a REST API at `<host>/api/v1/`. Auth is a bearer token — typically set from `FORGEJO_TOKEN` env. The token must have admin scope on the target repo.

## 1. Branch protection

Forgejo treats branch protection as a list of "protection rules," not a single object per branch. First check if a rule already exists for `main`, then POST or PATCH accordingly.

### List existing rules

```bash
curl -s -H "Authorization: token $FORGEJO_TOKEN" \
  "$FORGE_URL/api/v1/repos/$OWNER/$REPO/branch_protections"
```

If a rule with `rule_name: "main"` exists, PATCH it. Otherwise POST.

### Create rule

Endpoint: `POST /api/v1/repos/{owner}/{repo}/branch_protections`

```json
{
  "rule_name": "main",
  "enable_push": false,
  "enable_merge_whitelist": false,
  "required_approvals": 0,
  "block_on_rejected_reviews": true,
  "block_on_official_review_requests": true,
  "block_on_outdated_branch": false,
  "dismiss_stale_approvals": true,
  "require_signed_commits": false,
  "enable_status_check": true,
  "status_check_contexts": [
    "ci / build",
    "ci / test"
  ],
  "enable_approvals_whitelist": false,
  "enable_force_push": false
}
```

### Update rule

Endpoint: `PATCH /api/v1/repos/{owner}/{repo}/branch_protections/{rule_name}`

Same body shape as create.

### Invoke

```bash
curl -s -X POST \
  -H "Authorization: token $FORGEJO_TOKEN" \
  -H "Content-Type: application/json" \
  -d @payload.json \
  "$FORGE_URL/api/v1/repos/$OWNER/$REPO/branch_protections"
```

## Field notes

- `enable_push: false` blocks direct pushes — all changes must go through a PR.
- `block_on_outdated_branch: false` is the equivalent of GitHub's `strict: false`. Setting to `true` forces rebases before merge.
- `required_approvals: 0` combined with the next flag means approvals are optional but **rejections** block the merge (`block_on_rejected_reviews: true`). For solo repos, this is usually what you want.
- `status_check_contexts` comes from Forgejo Actions job IDs. The format depends on the workflow — typically `<workflow-name> / <job-id>`.
- `enable_force_push: false` disallows force-pushes even for admins. Keep this on.
- `require_signed_commits`: turn on only if all contributors have GPG keys set up; otherwise it silently blocks their commits.

## Discovering status check contexts on Forgejo

Forgejo Actions runs workflows from `.forgejo/workflows/*.yml` (or `.github/workflows/*.yml` if Forgejo is configured to read those). Each job produces a status check named `<workflow-name> / <job-id>`. The workflow name is the top-level `name:` field; the job id is the key under `jobs:`.

Example:

```yaml
name: ci
on: [push, pull_request]
jobs:
  build:
    runs-on: ubuntu-latest
    steps: ...
  test:
    runs-on: ubuntu-latest
    steps: ...
```

Produces two status check contexts: `ci / build` and `ci / test`.

To verify the exact strings, trigger one PR and read the status check list on the PR page.

## What is not available on Forgejo

- **Fork-PR contributor approval for Actions**: Forgejo Actions doesn't have GitHub's `fork-pr-contributor-approval` setting. Runner secrets are not automatically gated on PRs from forks, so **do not put sensitive secrets in Forgejo Actions** unless the repo is tightly controlled (private + trusted contributors only).
- **Private security advisories**: Forgejo has no equivalent of GitHub's private advisory flow. Use an email in `SECURITY.md` or direct vulnerability reports to a confidential issue tracker if the instance is configured for it.
- **Dependabot**: Not a Forgejo feature. Use Renovate — write `renovate.json` and run Renovate either as a scheduled Forgejo Actions job (using the Renovate action) or via the hosted bot at app.renovatebot.com if the Forgejo instance is internet-reachable.
