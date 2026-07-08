---
name: work-on-pr
description: Pick up an existing open GitHub pull request and set up to continue its development — fetch the PR with the gh CLI, create a Claude worktree checked out on the PR's actual branch, and read through its diff, description, commits, and review activity to get oriented. Takes the PR as an argument - a full URL, owner/repo#123, or a bare number resolved against the current repo. Stops after setup and orientation: no planning, no implementing. Use when the user drops a PR link and says "continue this PR", "pick up this PR", "work on PR #123", or similar.
---

<!--
FLOW RUNDOWN (quick reference)

1. Resolve the PR      — parse URL / owner-repo#N / bare number; fetch title,
                         body, state, branch, commits, reviews, comments,
                         checks via gh. Repo mismatch or non-open PR → stop
                         and check with the user.
2. Reuse or create worktree — `git worktree list --porcelain` first; if the
                         PR's branch is already checked out somewhere (e.g.
                         leftover from work-on-issue), EnterWorktree(path=…)
                         into it instead. Otherwise EnterWorktree with name
                         pr-<N>-<slug> (cuts a throwaway branch off default),
                         then `gh pr checkout <N>` inside it to switch to the
                         PR's real branch, then clean up the throwaway branch.
3. Familiarize         — read the diff, commit history, description, and
                         review/comment threads; check CI status. Summarize
                         what's done and what's outstanding.
4. Report and stop     — hand the summary and worktree location to the user.
                         No plan, no edits, no commits — that's a separate
                         next step the user asks for explicitly.

Load-bearing: check for an existing worktree on the PR's branch before
creating a new one — git refuses to check out the same branch twice, so
skipping this check breaks on any PR already picked up via work-on-issue.
When one must be created, it must land on the PR's actual branch, not a
fresh branch off default — EnterWorktree alone doesn't do this, `gh pr
checkout` inside it does. This skill ends at orientation; it does not plan
or implement.
-->

# Working on an existing pull request

Take a PR reference from argument to "ready to continue": fetch and understand the PR, check out its actual branch in an isolated worktree, and read enough of its diff, history, and review activity to know what's been built and what's left. Then stop — this skill only sets the stage; planning and implementing are the user's call to make next, potentially with a different skill.

**Shell state does not persist between commands.** Each Bash call runs in a fresh shell. Once a value is resolved (PR number, owner/repo, head branch), substitute the **literal** into later commands — don't rely on `export` or shell variables carrying over.

## 1. Resolve and inspect the PR

Accept the argument in any of these forms:

- Full URL — `https://github.com/OWNER/REPO/pull/123`
- Shorthand — `OWNER/REPO#123`
- Bare number — `123` or `#123`, resolved against the repo in the current working directory

No argument at all → ask the user for the PR link or number; don't guess from recently opened PRs.

Fetch everything needed to understand where the PR stands — reviews and comments included, since requested changes and unresolved discussion live there, not in the description:

```bash
gh pr view <URL-or-NUMBER> --json number,title,body,state,url,isDraft,headRefName,baseRefName,headRepositoryOwner,author,labels,comments,reviews,statusCheckRollup \
  --jq '{number, title, state, isDraft, url, head: .headRefName, base: .baseRefName, author: .author.login, labels: [.labels[].name], body, comments: [.comments[] | {author: .author.login, body}], reviews: [.reviews[] | {author: .author.login, state, body}], checks: [.statusCheckRollup[]? | {name, status, conclusion}]}'
```

Two checks before going further:

- **Repo match.** Compare the PR's repo against `gh repo view --json nameWithOwner --jq .nameWithOwner` for the current directory. If they differ, stop and tell the user — they're likely in the wrong checkout, and creating a worktree here would put the work in the wrong repository. Don't clone the other repo on your own initiative.
- **State.** If the PR is closed or merged, flag it and confirm the user really wants to continue it before doing anything else — it may have been superseded, or they may have pasted the wrong link. A draft PR is fine to proceed with; just note the draft status in your summary.

Summarize back what the PR is trying to do in a sentence or two before creating anything.

## 2. Reuse or create the worktree on the PR's branch

**Check for a leftover worktree first.** The PR's branch may already be checked out somewhere — e.g. a `work-on-issue` run that implemented this PR and left its worktree in place, or an earlier `work-on-pr` run on the same PR. Creating a second worktree for a branch that's already checked out elsewhere fails (git refuses to check out the same branch twice), so look before creating:

```bash
git worktree list --porcelain
```

Scan the output for an entry whose `branch` matches `refs/heads/<headRefName>` from step 1. If one exists, switch into it with **EnterWorktree**, passing its `path` — do not create a new worktree. Skip straight to fetching latest (`git fetch origin && git merge --ff-only origin/<headRefName>`, or note if the local branch has diverged) and then to step 3; the branch is already correct, so there's no placeholder to check out or clean up.

If no matching worktree exists, derive a name from the PR: `pr-<NUMBER>-<slug>`, where the slug is the title kebab-cased (lowercase, letters/digits/dashes only, trimmed so the whole name stays under 64 characters).

Create it with the **EnterWorktree** tool, passing that name. EnterWorktree always cuts a fresh branch off the default branch (or current HEAD) — it has no notion of the PR's actual branch, so that new branch is just a placeholder to get an isolated directory.

Once inside the worktree, switch it onto the PR's real branch:

```bash
gh pr checkout <NUMBER>
```

This fetches the PR's head (handling fork-owned branches transparently) and checks it out in place of the placeholder branch. Then clean up the placeholder — it's empty (cut from default, no edits made on it yet) and no longer checked out anywhere, so it's safe to remove:

```bash
git branch -D pr-<NUMBER>-<slug>
```

If EnterWorktree isn't available (a non-Claude agent running this skill), fall back to plain git — check `git worktree list` for an existing checkout of `<headRefName>` first and `cd` into it if found, otherwise:

```bash
git fetch origin
git worktree add .claude/worktrees/pr-<NUMBER>-<slug>
cd .claude/worktrees/pr-<NUMBER>-<slug>
gh pr checkout <NUMBER>
```

Do all subsequent work inside that directory, on the PR's own branch — not on a new branch, and not on the default branch.

## 3. Familiarize yourself with the PR

Read enough to explain the PR back accurately, without proposing changes yet:

- **The diff.** `gh pr diff <NUMBER>` for the full changeset, or `git diff <BASE>...HEAD` once checked out. Skim for shape and scope before reading any one file closely.
- **Commit history.** `git log <BASE>..HEAD --oneline` to see how the work was built up — WIP commits, reverts, and fixups often reveal what the author was struggling with or reconsidered.
- **Description and comments.** The PR body states intent; comments and reviews often narrow it, request changes, or flag known-incomplete parts. Treat the latest substantive comment or review as more current than the original body.
- **Review state.** Note any `CHANGES_REQUESTED` reviews and whether they've been addressed by later commits, and any unresolved review threads if `gh pr view --comments` or the web UI surfaces them.
- **CI status.** The `checks` pulled in step 1 — call out anything failing; it's likely the most immediate next thing to fix.

## 4. Report and stop

Summarize for the user, without proposing a plan:

- What the PR does and its current state (draft/ready, mergeable, CI status).
- What's already implemented, based on the diff and commits.
- What's outstanding — requested changes, unresolved comments, failing checks, or gaps between the description and the actual diff.
- The worktree location and branch name, so the user knows where to direct the next step.

Do not enter plan mode, propose an approach, or edit any files. If the user wants to keep going, that's a separate ask — implementing fixes, addressing review comments, or extending the PR each have their own shape and, in the case of addressing review comments, their own skill.

The worktree stays in place either way — the user decides at session end whether to keep or remove it.

## Done when

The PR was fetched and understood (description, comments, reviews, and CI status included) before anything was created; existing worktrees were checked and reused if the PR's branch was already checked out somewhere, otherwise a new one was created and switched onto the PR's actual head branch, not a fresh branch off default; the diff and commit history were read closely enough to summarize accurately what's done and what's outstanding; and the session stopped at that summary without entering planning or making any edits.
