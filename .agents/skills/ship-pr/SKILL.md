---
name: ship-pr
description: Use when pushing a branch, opening a pull request, waiting on CI or a review gate, addressing review findings, merging to the default branch, or promoting a change to staging or production in a GitHub repository with GitHub Actions. Also use when a PR seems stuck, silent, or shows no checks — that situation has a specific cause this skill diagnoses.
---

# Shipping a pull request

Drive a branch from push to production without dead waits and without
guessing what the repository requires. Each rule below exists because the
opposite behavior cost real hours: silent PRs that were never going to run,
gates read from the wrong signal, and green suites mistaken for live changes.

This skill is portable. It never assumes an owner, branch model, review bot,
or deploy target. Everything project-specific comes from the **shipping
profile** — read it first, or derive it.

## Step 0: load the shipping profile

1. Look for a `Shipping profile` section in the repository instructions
   (`AGENTS.md`, `CLAUDE.md`, or whatever the repository names as its agent
   instructions). When it exists, follow it; it wins over the defaults here.
2. When it does not exist, derive it with the discovery commands in
   [`references/shipping-profile.md`](references/shipping-profile.md). Do not
   invent a value you could not observe — write `unknown` and say which
   command would settle it.
3. When you derived it, offer to add the profile to the repository
   instructions in a `docs:` change, so the next session reads it instead of
   rediscovering it. Do not add it silently inside an unrelated PR.

The profile answers: repository identity, default branch, required status
checks, whether unresolved review threads block merge, allowed merge methods,
approval rules, review bots, local gates, migration tooling, and the release
path from merge to production.

Re-verify a stored profile when the evidence disagrees with it (a check name
that never appears, a merge refused for a reason it does not list). Rulesets
change; the profile is a cache, not the source of truth.

## The one diagnostic that comes before any wait

```bash
gh pr view <n> --json mergeable,mergeStateStatus
```

A PR that is `CONFLICTING`/`DIRTY` produces **no checks at all** — not
pending, not red, nothing. `pull_request` workflows run against the merge
ref; when it cannot be built, GitHub creates zero runs and fires zero
notifications. An empty Checks tab minutes after a push is a conflict until
proven otherwise. Waiting for a review that can never start hangs forever.

This is not a one-time check. The base moves while you work — a PR that was
`MERGEABLE` falls into conflict the moment an overlapping PR merges. Re-check
after every long wait and after every merge to the default branch that
touches files your branch touches.

When conflicting: rebase onto the default branch, resolve, run the local
gates, push. Checks appear on the new SHA by themselves.

Other `mergeStateStatus` values are diagnoses too: `BLOCKED` means a required
check, approval, or thread rule is unmet — find which one before waiting;
`BEHIND` matters only when the profile says the ruleset requires up-to-date
branches.

## The review gate, mechanically

Detect the gate the way the ruleset does — never by string-matching
comments:

```bash
# Required status checks, by conclusion
gh pr checks <n> --json name,bucket

# Unresolved review threads
gh api graphql -F owner=<owner> -F name=<repo> -F n=<n> -f query='
  query($owner: String!, $name: String!, $n: Int!) {
    repository(owner: $owner, name: $name) { pullRequest(number: $n) {
      reviewThreads(first: 100) { nodes { id isResolved
        comments(first: 1) { nodes { databaseId author { login } body } } } }
    } } }'
```

- Only the checks the profile lists as **required** block merge. Other
  checks are information; read them, but do not hold a merge on them.
- When the profile says threads must be resolved, **zero unresolved
  threads** is part of the gate even when the merge button is already green.
- AI review bots (PR-Agent, CodeRabbit, Copilot review, and similar) usually
  post findings as **inline review threads**, not as one summary comment.
  Grepping issue comments for a heading misses every finding and reports a
  gate that never closes.
- A bot that re-reviews on every push opens new threads after each fix
  commit. The loop converges; treat each round as normal, not scope creep.
  But each round is usually a full review of the **whole diff**, billed per
  token — see "Review rounds cost money" below.
- A bot check can be green while the review failed (for example a comment
  saying suggestions could not be generated). Read what it posted, not just
  the tick. A `cancel` conclusion is a killed run, not a pass — re-run it.
  `skipping` is a pass only where the workflow skips on purpose (for example
  promotion PRs); anywhere else it is suspicious.

## The findings loop

For each unresolved thread, do exactly one of:

1. **Apply** — fix, test, commit, push. Reply with the commit hash, then
   resolve:

```bash
gh api -X POST repos/<owner>/<repo>/pulls/<n>/comments/<root_comment_id>/replies \
  -f body="Applied in <sha> — <one line on what changed>."
gh api graphql -f query='mutation { resolveReviewThread(input:
  {threadId: "<thread_node_id>"}) { thread { isResolved } } }'
```

2. **Decline** — reply with the technical reason (an invariant that already
   provides the guarantee, a cost the suggestion would force for nothing),
   then resolve. Recording the why is what makes a decline reviewable;
   silent resolution is a violation.

Never resolve a thread you neither applied nor answered. Replying and
resolving without a push does not trigger a new bot review; a push does.

## Review rounds cost money

When an AI reviewer re-reviews on every push, the bill is
`rounds × whole diff`. Size multiplies every round, not just the first.

- **Iterate as a draft.** Open the PR with `gh pr create --draft` when the
  bot skips drafts (most do, or can be configured to). The draft saves the
  review bill, not CI: required checks still run on every push, and the
  local gates still apply. When the gates are green, `gh pr ready <n>` —
  that one event buys the first full review.
- **Keep PRs small.** Around 500 changed lines / 10 files unless the profile
  says otherwise. Bigger than that, split by task before opening; reviewers,
  human or not, read a slice better than a story.
- **Batch fixes.** After ready, apply a round's findings in one push rather
  than one push per finding.

### Stop the loop on purpose — the round-5 flip

The loop ends only when a round finds nothing, and the reviewer decides
that, not you. Applying every nit can take a PR to 20+ rounds.
**Resolving costs nothing; pushing costs a full re-review.** That asymmetry
is the decision:

- Rounds 1–4: apply what is real, batched one push per round.
- **From round 5:** decline, in one batch and with the reason recorded, every
  low-importance finding that has no reachable failure scenario on code that
  passes the gates and its tests — then merge. Saying the cost out loud in
  the reply is a legitimate engineering reason.
- A finding whose fix produces the next finding is one design conversation:
  resolve the whole shape in a single push instead of trading turns.
- A minor finding **rides along free** when a round is already being bought
  for something bigger. The rule refuses to spend a round on a nit, not to
  fix one.

When the profile lists a run budget for the review bot (for example a
repository variable that stops automatic review after N runs per branch),
treat the budget as spent effort, not as a failure: apply or decline the
open threads, and ask for another round explicitly (`/review`) only when it
is genuinely warranted.

## Flaky checks are a budget, not a loop

Rerun a failed check **at most twice**:

```bash
gh run rerun <run_id> --repo <owner>/<repo> --failed
```

Before rerunning, read the failure (`gh run view <run_id> --log-failed`) and
distinguish infrastructure signatures (network timeouts to package
registries or api.github.com, runner or container start-up errors) from real
failures. On the third failure of the same test, stop rerunning: if the
check is not required and the failing test is untouched by the diff, note it
in the PR and move on — recurring flake is a backlog item. A failing
**required** check is never waved off as flake; diagnose it.

## Waits never serialize independent work

While one PR's checks run, do other unblocked work: fix another PR's
findings, rebase the next branch, write the docs. Watch in the background
and act on events. A watcher built from the profile:

```bash
REQ='["<required-check-1>","<required-check-2>"]'   # from the profile
while true; do
  mg=$(gh pr view <n> --json mergeable --jq .mergeable)
  [ "$mg" = "CONFLICTING" ] && { echo CONFLICT; break; }
  ok=$(gh pr checks <n> --json name,bucket | jq --argjson req "$REQ" \
    '[.[] | select(.name as $c | $req | index($c))]
       | length > 0 and all(.bucket=="pass" or .bucket=="skipping")')
  thr=$(gh api graphql ... --jq '[.. | select(.isResolved? == false)] | length')
  [ "$ok" = "true" ] && [ "$thr" = "0" ] && { echo READY; break; }
  sleep 60
done
```

Drop the thread condition when the profile says threads do not block merge.
`gh pr checks` exits non-zero while checks are pending or failing; read its
JSON, not its exit code.

If two PRs are gated and one is ready, merge the ready one now — do not hold
it hostage to a sibling. Merging it may throw the sibling into conflict; that
is expected, cheaper than the serialized wait, and the conflict guard catches
it.

## Before the PR: the local contract

- Run **every** local gate the profile lists (normally the same commands the
  required CI job runs), in a clean checkout or worktree, never a subset. A
  gate you skipped is a gate CI will run for you, slower.
- Use the repository's PR template when it has one, and its commit
  convention (for example Conventional Commits). Follow its attribution rules
  for commit trailers and PR bodies — some rulesets block merges from
  commits whose trailers name an unknown co-author.
- New migration: after **every** rebase, check the migrations directory for a
  version or prefix collision. Same-number files merge cleanly in git and
  break apply order silently.
- Schema before code: when the release path deploys code on merge, apply the
  migration to the target environment **before** merging code that needs it,
  and confirm it is applied with the tool's own status command. Code ahead of
  schema is an outage, not a warning. A green migrations workflow is not proof
  a database was touched — read the job that applied it.
- Never bypass the gate (`gh pr merge --admin`, disabling a ruleset) to land
  a change. If the gate is wrong, fix the gate in its own PR.

## Merge

```bash
gh pr merge <n> --repo <owner>/<repo> --<merge-method>
```

Use a merge method the profile allows, matching the history already on the
default branch. After the merge, clean up: remove the branch's worktree and
delete the branch locally and on the remote.

## Promote and prove it live

Follow the release path in the profile. The common shapes:

- **Merge deploys.** A Git-integrated host or a deploy workflow ships every
  merge to the default branch. The merge *is* the release: everything in
  "Before the PR" must be true before you press it.
- **Promotion branches** (for example default → `staging` → `production`).
  Long-lived promotion branches usually diverge; promote with a **merge
  commit**, not fast-forward-only. Promote to production only after the
  change is proven live on staging.
- **Tags or releases.** A tag or a semantic-release run publishes; a
  promotion carrying only `docs:`/`chore:`/`ci:` commits may correctly end in
  "no release".

In every shape, a green suite is not a live endpoint. Prove the change is
running: the deployment points at the merged commit SHA (the host's
deployment API or the Deployments tab), and the user-visible behavior or
endpoint responds as intended. Report what you observed, not that the merge
succeeded.

## Anti-patterns, named

- **Waiting on a silent PR** — silence is a conflict, not progress.
- **Grepping comments for the gate** — required checks and threads are the
  gate.
- **Hard-coding the gate** — read it from the profile or the ruleset; check
  names, bots, and branches differ per repository.
- **Serial gate-watching** — act on whatever is unblocked now.
- **Infinite flake reruns** — two, then classify and move on.
- **Merging seconds after the first comment** — the gate is every required
  check green and, when required, zero unresolved threads.
- **Pushing per finding, or applying every nit forever** — batch each round,
  and flip to decline-and-merge from round 5.
- **Calling a merge a release** — prove the deployed SHA and the behavior.
