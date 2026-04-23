# skills

A library of [Claude Code](https://claude.com/claude-code) skills.

Skills are Markdown-driven capabilities that Claude Code loads on demand. Each skill lives in its own directory with a `SKILL.md` entry point, optional `templates/`, and optional `references/` loaded lazily.

## Installed skills

- **[repo-bootstrap](./repo-bootstrap/)** — Apply a baseline of security controls and governance files to a new or under-protected GitHub repo. Supports apply mode, audit mode (read-only scoreboard), and dry-run mode. Covers license, security policy, CODEOWNERS, `.gitignore`, Dependabot, branch protection, secret scanning, Dependabot alerts, fork-PR approval gating, default workflow permissions, private advisory intake. Optional CI hardening templates for Chainguard Dockerfile, Grype image scanning with Sigstore SBOM attestation, `pinact` SHA-pin enforcement, and runner hardening via StepSecurity.
- **[workflow-surgery](./workflow-surgery/)** — Edit GitHub Actions workflows and other YAML in place, or propagate a change across many repos. Covers the tool ladder (Edit → yq → perl → sed), multi-line regex gotchas, the Contents API no-clone fan-out route and its signing tradeoff, and the clone-and-sign fallback when personal commit signatures matter.

## Installing a skill

Claude Code loads skills from `~/.claude/skills/`. To install a skill from this repo:

```bash
git clone https://github.com/wingnut128/skills.git /tmp/skills
cp -R /tmp/skills/<skill-name> ~/.claude/skills/
```

Or symlink if you want live updates as you `git pull`:

```bash
ln -s "$(pwd)/<skill-name>" ~/.claude/skills/<skill-name>
```

## Contributing

Pull requests welcome. Each skill must:

- Have a `SKILL.md` with a clear `description` frontmatter line (first-person voice — the description is what Claude matches against triggers).
- Be idempotent where it performs stateful actions.
- Document placeholders (`{{LIKE_THIS}}`) and substitution expectations.
- Pass the `pin-check` workflow once CI is wired (all `uses:` lines pinned by SHA).

## License

MIT — see [LICENSE](./LICENSE).
