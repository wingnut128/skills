# Stack matrix

Which templates to combine for a given primary stack.

| Stack   | Detect                                  | .gitignore extra      | Dep config             |
|---------|-----------------------------------------|-----------------------|------------------------|
| bun     | `package.json` + `bun.lock` or `bunfig` | `gitignore-bun`       | `dependabot-bun.yml`   |
| node    | `package.json` + no bun lock            | `gitignore-node`      | `dependabot-node.yml`  |
| rust    | `Cargo.toml`                            | `gitignore-rust`      | `dependabot-rust.yml`  |
| python  | `pyproject.toml` or `requirements.txt`  | `gitignore-python`    | `dependabot-python.yml`|
| go      | `go.mod`                                | `gitignore-go`        | `dependabot-go.yml`    |
| mixed   | multiple of the above                   | concat applicable     | concat applicable      |

## Always applied

- `gitignore-security-base` — prepend to every repo's `.gitignore` regardless of stack.
- License + SECURITY.md + CODEOWNERS + CLAUDE.md.stub — identical across stacks, only parametrized.

## License-specific notes

- `LICENSE-MIT.txt` and `LICENSE-BSD-3-Clause.txt` are embedded verbatim with `{{YEAR}}` and `{{OWNER}}` substitutions.
- `LICENSE-Apache-2.0.txt` is a stub containing `{{FETCH_APACHE_2_0}}`. When the user picks Apache-2.0, fetch the canonical text from `https://www.apache.org/licenses/LICENSE-2.0.txt` (stable URL, verbatim license body) and substitute it in — do not attempt to hand-write it. If offline, error out rather than writing a guess.

## Mixed-stack repos

If the repo has multiple stacks (e.g. a Python service with a React frontend), concatenate the per-stack `.gitignore` files and emit multiple `package-ecosystem:` blocks in a single `dependabot.yml`. The user will occasionally want the opposite (keep the dependabot config lean, only update the primary stack) — ask them.
