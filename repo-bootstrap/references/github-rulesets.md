# GitHub rulesets — upgrade path from classic protection

Repository rulesets are GitHub's newer branch/tag/push governance system. They coexist with classic branch protection — both are evaluated, and the union of all rules applies. This reference covers the payload shape and when to prefer rulesets. The default `repo-bootstrap` path still uses classic protection (simpler, well-documented); use rulesets when you need features classic doesn't offer.

## When to prefer rulesets over classic

| Need | Classic | Rulesets |
|---|---|---|
| Require N approving reviews on a branch | ✅ | ✅ |
| Required status checks | ✅ | ✅ |
| Enforce admins | ✅ (boolean) | ✅ (via `bypass_actors`, per-actor) |
| Dismiss stale reviews | ✅ | ✅ |
| Require code-owner review | ✅ | ✅ |
| Conversation resolution required | ✅ | ✅ |
| Forbid force-push / deletion | ✅ | ✅ |
| **Per-actor bypass allowances** (e.g. only owner bypasses review count) | ❌ | ✅ |
| **Target multiple branches with one rule** (`main`, `release/*`) | ❌ (one protection per branch) | ✅ (fnmatch include/exclude) |
| **Required signed commits** | ❌ (repo-wide only) | ✅ (per-branch) |
| **Push rulesets** (rules that apply to direct pushes, not just PR merges) | ❌ | ✅ |
| **Required deployment environments before merge** | ❌ | ✅ |
| **Block commit messages matching a regex** | ❌ | ✅ |
| Simple API, fewer endpoints | ✅ | ❌ (one `/rulesets` endpoint but denser payload) |
| UI discoverability for casual users | ✅ (Settings → Branches) | ⚠️ (Settings → Rules, newer UI) |

**Default for repo-bootstrap:** classic protection. Switch to rulesets when:

- You have a solo-owner repo and want yourself to bypass the review-count rule without disabling `enforce_admins` globally
- You govern multiple release branches from one rule (e.g. `main`, `release/*`, `hotfix/*`)
- You need required signed commits on specific branches
- You need push rulesets (rules that don't require a PR — e.g. "never push directly to main")

## Endpoint

```text
POST /repos/{owner}/{repo}/rulesets
GET  /repos/{owner}/{repo}/rulesets
GET  /repos/{owner}/{repo}/rulesets/{ruleset_id}
PUT  /repos/{owner}/{repo}/rulesets/{ruleset_id}
DELETE /repos/{owner}/{repo}/rulesets/{ruleset_id}
```

## Full example: equivalent of the repo-bootstrap classic baseline

This ruleset matches what the classic `repo-bootstrap` applies, plus one feature classic can't express (solo-owner bypass on review count):

```json
{
  "name": "main branch baseline",
  "target": "branch",
  "enforcement": "active",
  "bypass_actors": [
    {
      "actor_id": 5,
      "actor_type": "RepositoryRole",
      "bypass_mode": "pull_request"
    }
  ],
  "conditions": {
    "ref_name": {
      "include": ["~DEFAULT_BRANCH"],
      "exclude": []
    }
  },
  "rules": [
    {
      "type": "deletion"
    },
    {
      "type": "non_fast_forward"
    },
    {
      "type": "required_linear_history"
    },
    {
      "type": "pull_request",
      "parameters": {
        "required_approving_review_count": 1,
        "dismiss_stale_reviews_on_push": true,
        "require_code_owner_review": true,
        "require_last_push_approval": false,
        "required_review_thread_resolution": true
      }
    },
    {
      "type": "required_status_checks",
      "parameters": {
        "required_status_checks": [
          { "context": "rustfmt" },
          { "context": "clippy" },
          { "context": "test (ubuntu-24.04)" }
        ],
        "strict_required_status_checks_policy": false
      }
    }
  ]
}
```

**Key fields:**

- `target`: `branch` | `tag` | `push`. `push` rulesets apply to direct pushes (no PR); useful for "block any push to main" or commit-message patterns on every push.
- `enforcement`: `active` (rules apply) | `evaluate` (dry-run — logs violations but allows them, useful for rollout) | `disabled`.
- `conditions.ref_name.include`: fnmatch patterns (`main`, `release/*`, `refs/heads/feature/**`) or the special `~DEFAULT_BRANCH` / `~ALL` tokens.
- `bypass_actors[].actor_type`: `RepositoryRole` | `Team` | `Integration` | `OrganizationAdmin`. Role IDs: `1`=read, `2`=triage, `3`=write, `4`=maintain, `5`=admin.
- `bypass_actors[].bypass_mode`: `always` (bypass in any context) | `pull_request` (bypass only when merging a PR, not on direct push). **Prefer `pull_request`** — it keeps the PR workflow intact while letting the bypasser merge without meeting every rule.

## Rule types

| Type | Parameters | Purpose |
|---|---|---|
| `creation` | — | Block branch creation matching the rule. |
| `update` | — | Block pushes that update existing commits. |
| `deletion` | — | Block branch deletion. |
| `required_linear_history` | — | Require fast-forward or rebase; block merge commits. |
| `non_fast_forward` | — | Block force-push. |
| `required_deployments` | `required_deployment_environments: [string]` | Require successful deployment to listed environments before merge. |
| `required_signatures` | — | Require signed commits on this branch. |
| `pull_request` | `required_approving_review_count`, `dismiss_stale_reviews_on_push`, `require_code_owner_review`, `require_last_push_approval`, `required_review_thread_resolution` | PR gate. |
| `required_status_checks` | `required_status_checks: [{context, integration_id?}]`, `strict_required_status_checks_policy` | CI gate. |
| `commit_message_pattern` | `name`, `pattern`, `operator: starts_with \| ends_with \| contains \| regex`, `negate` | Enforce/forbid commit message shapes. |
| `commit_author_email_pattern` | same as above | Restrict author email. |
| `committer_email_pattern` | same as above | Restrict committer email. |
| `branch_name_pattern` | same as above | Enforce branch naming. |
| `tag_name_pattern` | same as above | Enforce tag naming. |

## Push rulesets

`target: "push"` is the most distinctive rulesets feature. Applies to **every push** to matching refs, including direct pushes (not just PR merges). Use cases:

- **Forbid direct pushes to main:** `target: push`, `conditions.ref_name.include: ["~DEFAULT_BRANCH"]`, `rules: [{type: "update"}]`. Forces every change through a PR.
- **Enforce commit message format repo-wide:** `target: push`, `conditions.ref_name.include: ["~ALL"]`, `rules: [{type: "commit_message_pattern", parameters: {pattern: "^(feat|fix|docs|chore)(\\(.+\\))?: ", operator: "regex"}}]`.

Push rulesets don't have PR-specific rule types (`pull_request`, `required_status_checks`) — those only make sense in a merge context.

## Coexistence with classic branch protection

**Both systems evaluate independently** and the **union** of their rules applies. If classic protection requires status check `X` and a ruleset requires status check `Y`, a PR needs both.

**Don't remove classic protection when adding a ruleset** — the skill's default path (classic) is still in force. Removing classic to "simplify" before the ruleset is validated can silently drop required checks.

**Safe migration shape (not implemented by this skill — see Stage E in TODO.md):**

1. Add the ruleset in `enforcement: evaluate` mode. Watch for violations in `Settings → Rules → <ruleset> → Insights` for a few PR cycles.
2. Flip to `enforcement: active`.
3. Confirm the ruleset enforces every control the classic protection does.
4. `DELETE /repos/{owner}/{repo}/branches/{branch}/protection` to remove the classic rule.
5. Keep a rollback path: save the classic protection JSON payload before deleting, so a `PUT` restores it if the ruleset misbehaves.

## Verification

```bash
# List all rulesets
gh api "repos/$OWNER/$REPO/rulesets"

# Read a specific ruleset
gh api "repos/$OWNER/$REPO/rulesets/$RULESET_ID"

# List rules from rulesets that apply to a branch (rulesets only — does NOT include classic protection)
gh api "repos/$OWNER/$REPO/rules/branches/$BRANCH"
```

**Important:** `/rules/branches/{branch}` returns **only ruleset rules**, not classic branch protection. On a repo with classic protection active and no rulesets, this endpoint returns `[]`. Don't treat an empty response as "no rules on this branch" — classic protection may still be active. To see the full picture, query both `/branches/{branch}/protection` (classic) and `/rules/branches/{branch}` (rulesets).

## Skip conditions

- **User-owned repos:** all ruleset features work, but `OrganizationAdmin` bypass_actor type is meaningless (no org exists).
- **Enterprise rulesets** (`POST /enterprises/{enterprise}/rulesets`) exist for GHEC but are out of scope here.
