---
name: workflow-surgery
description: >-
  Use when editing GitHub Actions workflows or other YAML in place — adding
  steps, pinning actions, or propagating a change across many repos.
  Invoke for asks like "add a step to this workflow", "SHA-pin these
  actions", "propagate this change across my repos", or any multi-line
  YAML rewrite Edit can't express cleanly.
---

# workflow-surgery

YAML edits where indentation, multi-line blocks, and comments all matter. This skill picks the right tool and calls out the footguns.

## Tool ladder

Try in order; stop at the first fit.

1. **`Edit`** — default. If `old_string` is unique, use it.
2. **`yq`** — structural edits (`.jobs.test.steps += [...]`). Understands YAML. Limit: may reformat and strip some comments; diff before committing. Use **mikefarah/yq** (Go), not `python-yq`.
3. **`perl -0pe`** — textual multi-line rewrites where `yq` loses comments or can't express the pattern. Put non-trivial substitutions in a `.pl` file — shell-quoting complex regex inline breaks fast.
4. **`sed`** — single-line only; BSD/GNU `-i` differ. Prefer `perl -i`.

## Gotchas

- **Perl `\s` vs `[[:space:]]`** — when `\s` silently fails to match YAML indentation, swap to POSIX class first.
- **Indentation in `run: |` is content.** 6 vs 8 spaces is a real change.
- **Preserve `# vX.Y.Z` pin comments** on `uses:` lines — they document the pinned version.
- **`uses:` with matrix `os`** — disk cleanup steps are Linux-only; gate with `if: runner.os == 'Linux'`.

## Fan-out across repos

Two paths, different tradeoffs:

### API route (no clone)

```bash
# Fetch (save blob SHA — required for PUT)
gh api "repos/O/R/contents/PATH?ref=BRANCH" --jq '.sha' > /tmp/sha
gh api "repos/O/R/contents/PATH?ref=BRANCH" --jq '.content' | base64 -d > /tmp/f

# Transform
perl /tmp/patch.pl /tmp/f

# Branch + commit via PUT
BASE=$(gh api "repos/O/R/branches/BRANCH" --jq '.commit.sha')
gh api --method POST "repos/O/R/git/refs" -f ref="refs/heads/NEW" -f sha="$BASE"
gh api --method PUT "repos/O/R/contents/PATH" \
  -f message="msg" -f content="$(base64 -i /tmp/f)" \
  -f branch=NEW -f sha="$(cat /tmp/sha)"

gh pr create --repo O/R --head NEW --base BRANCH --title "..." --body "..."
```

Caveats:

- One file per PUT (multi-file → Git Data API or multiple commits).
- **Commits are unsigned.** PAT-authored commits via Contents API don't get signed — no web-flow signature, no personal signature. Only web-UI edits and GitHub Apps with commit-signing produce verified commits via API. If signature matters, use the clone route.

### Clone route

```bash
gh repo clone O/R -- --depth=1   # may fall back to SSH and hang — use HTTPS if so
cd R && git checkout -b NEW
# edit files
git commit -S -m "..."            # requires commit.gpgsign + signing key set up
git push -u origin NEW
gh pr create --title "..." --body "..."
```

Use this when signed commits matter, or for small N where the clone cost is trivial.

**SSH hang fallback:** if `gh repo clone` stalls, `git clone https://github.com/O/R.git` directly. Set `git remote set-url origin https://...` before push to avoid a second SSH attempt.

## Scope discipline

When adding a step to a workflow, target only the jobs that need it. A 10-second cleanup step in 8 jobs wastes 80 seconds per run. Check each job's actual resource usage before blanket-applying.

PR bodies for CI changes: **what / why / scope** — one line each.

## Anti-patterns

- Cloning to change one line (if signature isn't needed).
- `sed -i ''` on Linux (BSD syntax).
- `yq` on comment-heavy files without diffing the result.
- Bash `for` over a space-string (`files="a.yml b.yml"`) — use an array: `files=(a.yml b.yml); "${files[@]}"`.
- `perl` when `Edit` would work — slower, more escaping, no diff summary.
- Complex regex inline in shell — put it in a `.pl` file.
