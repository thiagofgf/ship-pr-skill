# Shipping profile

The shipping profile is the per-repository data the `ship-pr` skill needs.
Store it once in the repository's agent instructions (for example a
`## Shipping profile` section in `AGENTS.md`) so every session reads the same
facts. Derive it with the commands below; write `unknown` for anything you
could not observe, and name the command that would settle it.

## Template

```markdown
## Shipping profile

- **Repository:** <owner>/<repo>, default branch `<branch>`
- **Required checks:** `<check>`, `<check>` (ruleset or branch protection
  `<name or id>`)
- **Unresolved threads block merge:** yes | no
- **Approvals required:** <n>; code-owner review: yes | no
- **Merge methods:** merge | squash | rebase — history uses <method>
- **Review bots:** <name, where it is configured, when it is skipped (drafts?
  promotion PRs?)> | none
- **Review budget:** <run cap per branch and where it is set; PR size
  ceiling> | none
- **Local gates:** `<command>` && `<command>` … (mirror of the required job)
- **Migrations:** <tool and directory>; applied by <workflow | hand>; status
  command `<command>` | none
- **Release path:** <merge deploys to production via ...> | <default →
  staging → production promotion branches> | <tags/releases via ...>
- **Proof of deploy:** <how to read the deployed commit SHA>
- **Commit and PR rules:** <commit convention, PR templates, attribution
  rules>
```

## Discovery commands

Run them from the repository checkout. `gh` must be authenticated.

```bash
# Identity and default branch
gh repo view --json nameWithOwner,defaultBranchRef \
  --jq '"\(.nameWithOwner) \(.defaultBranchRef.name)"'

# Effective rules on the default branch (rulesets + branch protection)
gh api repos/<owner>/<repo>/rules/branches/<branch> \
  --jq '.[] | {type, parameters}'
#   required_status_checks.required_status_checks[].context -> required checks
#   pull_request.required_review_thread_resolution         -> threads block?
#   pull_request.required_approving_review_count           -> approvals
#   pull_request.allowed_merge_methods                     -> merge methods
#   pull_request.require_extra_approval_for_unattributed_changes -> attribution
#     rules matter for commit trailers

# Classic branch protection, when no ruleset applies (404 means none)
gh api repos/<owner>/<repo>/branches/<branch>/protection

# Workflows: CI jobs (their job names are the check names) and deploy jobs
ls .github/workflows/
grep -n "name:\|on:\|run:\|environment:" .github/workflows/*.yml

# Review bots: config files and who reviewed recent PRs
ls .pr_agent.toml .coderabbit.yaml .github/copilot* 2>/dev/null
gh pr list --state merged --limit 5 --json number --jq '.[].number' |
  xargs -I{} gh api repos/<owner>/<repo>/pulls/{}/reviews --jq '.[].user.login' |
  sort | uniq -c

# Merge method in use
git log --first-parent --format='%s' -20 <branch>

# Release path: promotion branches, environments, deployments
gh api repos/<owner>/<repo>/branches --jq '.[].name'
gh api repos/<owner>/<repo>/environments --jq '.environments[].name'
gh api "repos/<owner>/<repo>/deployments?per_page=5" \
  --jq '.[] | "\(.environment) \(.ref) \(.sha[:7]) \(.creator.login)"'

# Migration tooling (look for the directory the project actually uses)
ls -d supabase/migrations prisma/migrations db/migrate migrations alembic 2>/dev/null
```

Local gates: read the commands from the required CI job's `run:` steps and
from the PR template's checklist. When the two disagree (for example a
stricter `lint:ci` in CI than `lint` in the template), the CI command is the
gate.

## Reading the evidence

- A required check name is the **job name** (or `name:`) the ruleset lists,
  not the workflow file name. A check that the ruleset requires but no
  workflow produces blocks every PR forever — report it.
- Deployments created by a Git-integrated host (Vercel, Netlify, Render, and
  similar) on the default branch mean **merge deploys**. Treat the merge as
  the release.
- Long-lived branches named like `staging`, `production`, or `release/*`
  plus workflows triggered by pushes to them mean **promotion branches**.
- `on: push: tags` or a semantic-release step means **tags or releases**.
