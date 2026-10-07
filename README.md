# Ship PR Agent Skill

A portable [Agent Skill](https://agentskills.io/specification) that drives a
GitHub pull request from push to production: conflict diagnosis, the review
gate (required checks and review threads), the findings loop, flaky-check
budgets, background watching, merge, promotion, and proof that the change is
live.

It works in any repository hosted on GitHub with GitHub Actions. Nothing about
your project is hard-coded: the skill reads a **shipping profile** from your
agent instructions, or derives it from the GitHub API (rulesets, workflows,
review bots, deployments) and offers to record it.

## Skills

- `ship-pr`: push, open, gate, address findings, merge, promote, and verify a
  pull request using the repository's own rules.

## Install

### Codex

Ask Codex to install the skill from this repository, or run:

```bash
python "${CODEX_HOME:-$HOME/.codex}/skills/.system/skill-installer/scripts/install-skill-from-github.py" \
  --repo thiagofgf/ship-pr-skill \
  --path .agents/skills/ship-pr
```

The skill becomes available in the next Codex turn.

### Claude Code

Clone once and link the skill into the user-level discovery directory:

```bash
git clone https://github.com/thiagofgf/ship-pr-skill.git ~/.local/share/ship-pr-skill
mkdir -p ~/.claude/skills
ln -s ~/.local/share/ship-pr-skill/.agents/skills/ship-pr ~/.claude/skills/ship-pr
```

### Repository-local agents

Copy `.agents/skills/ship-pr` into the repository. For Claude Code
compatibility, link `.claude/skills` to `../.agents/skills`.

## Adapting it to a repository

Add a `## Shipping profile` section to the repository's `AGENTS.md` (or
equivalent) using the template in
[`references/shipping-profile.md`](.agents/skills/ship-pr/references/shipping-profile.md).
The same file lists the discovery commands the skill runs when no profile
exists. Typical values:

- required status checks and whether unresolved threads block merge;
- review bots and when they skip;
- local gates that mirror the required CI job;
- migration tooling and whether migrations are applied by a workflow or by
  hand;
- the release path: merge deploys, promotion branches, or tags.

## Requirements

- [GitHub CLI](https://cli.github.com/) (`gh`), authenticated with access to
  the repository.
- `jq` for the background watcher.

## Layout

```text
.agents/skills/
└── ship-pr/
    ├── SKILL.md
    └── references/shipping-profile.md
.claude/skills -> ../.agents/skills
```

`.agents/skills` is the only canonical source; the Claude path is a symlink so
the skill definition cannot drift.

## License

Distributed under the [MIT License](LICENSE).
