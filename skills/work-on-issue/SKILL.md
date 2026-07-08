---
name: work-on-issue
description: Pick up a GitHub issue and start working on it in an isolated worktree — fetch the issue with the gh CLI, create a Claude worktree named after it, explore the codebase in plan mode, get the plan approved, then implement. Takes the issue as an argument - a full URL, owner/repo#123, or a bare number resolved against the current repo. Use when the user drops an issue link and says "work on this", "pick up this issue", "start on #123", or similar.
---

<!--
FLOW RUNDOWN (quick reference)

1. Resolve the issue   — parse URL / owner-repo#N / bare number; fetch title,
                         body, state, labels, comments via gh. Repo mismatch or
                         closed issue → stop and check with the user.
2. Create worktree     — EnterWorktree with name issue-<N>-<slug>. Fallback for
                         non-Claude agents: git worktree add off the default branch.
3. Ask for more context — hard stop: end the turn and wait for an actual
                         reply before planning starts; blank reply means
                         proceed with the issue as written.
4. Plan                — EnterPlanMode; reconcile issue text against the actual
                         code (issues go stale); AskUserQuestion for genuine
                         ambiguity; plan cites issue #N; ExitPlanMode to approve.
5. Implement           — per approved plan, in the worktree, project checks pass.
6. Wrap up             — report; commit/push/PR ("Fixes #N") only on explicit
                         confirmation — those are outward-facing.

Load-bearing: fetch the issue BEFORE creating the worktree (the number and title
name it); plan approval gates implementation; nothing is pushed without a go-ahead.
-->

# Working on a GitHub issue

Take an issue reference from argument to implemented change: fetch and understand the issue, isolate the work in its own worktree, design the approach in plan mode with the user's sign-off, then build it. The worktree keeps the user's current checkout untouched — they can keep working on `main` while this task proceeds in parallel.

**Shell state does not persist between commands.** Each Bash call runs in a fresh shell. Once a value is resolved (issue number, owner/repo, default branch), substitute the **literal** into later commands — don't rely on `export` or shell variables carrying over.

## 1. Resolve and inspect the issue

Accept the argument in any of these forms:

- Full URL — `https://github.com/OWNER/REPO/issues/123`
- Shorthand — `OWNER/REPO#123`
- Bare number — `123` or `#123`, resolved against the repo in the current working directory

No argument at all → ask the user for the issue link or number; don't guess from recent issues.

Fetch everything needed to understand the task — comments included, since clarifications and scope changes often live there, not in the body:

```bash
gh issue view <URL-or-NUMBER> --json number,title,body,state,url,labels,assignees,comments \
  --jq '{number, title, state, url, labels: [.labels[].name], assignees: [.assignees[].login], body, comments: [.comments[] | {author: .author.login, body}]}'
```

Two checks before going further:

- **Repo match.** Compare the issue's repo against `gh repo view --json nameWithOwner --jq .nameWithOwner` for the current directory. If they differ, stop and tell the user — they're likely in the wrong checkout, and creating a worktree here would put the work in the wrong repository. Don't clone the other repo on your own initiative.
- **State.** If the issue is closed, flag it and confirm the user really wants to work on it before continuing — it may already be fixed, or they may have pasted the wrong link.

Summarize back what the issue asks for in a sentence or two before creating anything — if the issue is vague ("app is slow"), that's a planning problem to solve in step 4, not a reason to stop here.

## 2. Create the worktree

Derive a name from the issue: `issue-<NUMBER>-<slug>`, where the slug is the title kebab-cased (lowercase, letters/digits/dashes only, trimmed so the whole name stays under 64 characters — e.g. issue #482 "Login form loses state on refresh" → `issue-482-login-form-loses-state`).

Create it with the **EnterWorktree** tool, passing that name. This creates the worktree under `.claude/worktrees/` on a new branch off the default branch and switches the session into it — all subsequent exploration and edits happen there.

If EnterWorktree isn't available (a non-Claude agent running this skill), fall back to plain git — resolve the default branch first, then substitute literals:

```bash
git fetch origin
git worktree add .claude/worktrees/issue-<NUMBER>-<slug> -b issue-<NUMBER>-<slug> origin/<DEFAULT_BRANCH>
```

and do all subsequent work inside that directory.

## 3. Ask for more context

Before planning, give the user one chance to add anything about the issue that isn't captured in its body or comments — internal discussion, priority, constraints, a preferred approach, or things tried already. Ask directly, e.g.: "Anything else about this issue I should know before planning — internal context, constraints, preferred approach? Leave it blank if the issue as written covers it."

**This is a hard stop, not a rhetorical aside.** End your turn on this question and wait for the user's actual reply — do not answer it yourself, assume a blank reply, or continue into step 4 in the same turn. This applies even under a general bias toward not pausing for clarifying questions: that bias is for decisions you can make with a sensible default, not for skipping a step this skill defines as an explicit checkpoint.

Once the user replies, a blank or "no" answer means proceed with the issue as written — don't press further for input they've already declined to give. Fold anything they do provide into the plan in step 4 alongside the issue body and comments.

## 4. Plan the implementation

Enter plan mode with **EnterPlanMode** — issue-driven work is exactly the case for it: the requirements were written by someone else, possibly a while ago, and need reconciling against the code as it exists now.

While planning:

- **Verify the issue against reality.** Reproduce the bug or locate the feature area before designing anything. Issues go stale — the code may have moved, the bug may be half-fixed, the proposed solution in the issue may target code that no longer exists. Where the issue's description and the code disagree, the code wins; note the discrepancy in the plan.
- **Mine the comments.** Maintainer replies often narrow scope, reject approaches, or add acceptance criteria that supersede the original body. Treat the latest substantive maintainer comment as the current spec — unless the user's own input (step 3, or anything they say during planning) contradicts it.
- **The user outranks the paper trail.** The issue body and its comments are a fixed, possibly stale record; the user talking to you right now knows the current state of things. Where what the user says conflicts with the issue text or comments — priority, intended behavior, scope, an approach the issue proposed but the user now says is wrong — go with the user and note in the plan that this diverges from the written issue.
- **Ask, don't assume.** For genuine forks in the road — two valid architectures, unclear scope boundary, a suggested fix in the issue that you'd diverge from — use AskUserQuestion before finalizing the plan, so the plan the user approves is unambiguous.
- **Cite the issue.** The plan should reference `#<NUMBER>` and state which acceptance criteria (explicit or inferred) it satisfies, so the user can judge the plan against the issue, not just against your reading of it.

Then present the plan for approval with **ExitPlanMode**. Don't start editing files before the plan is approved.

## 5. Implement

Work the approved plan inside the worktree. Stay on the worktree's branch — never the default branch.

Run the project's checks as you go — consult `CLAUDE.md`, `README`, or package scripts for the right commands (test runner, linter, type-checker). If implementation reveals the approved plan doesn't survive contact with the code, stop and tell the user what changed rather than silently building something different from what they approved.

## 6. Wrap up

Report what was built against the issue's acceptance criteria, with the diff stat and check results — the actual outcome, not intent.

Committing, pushing, and opening a PR are outward-facing; offer them, but only act on explicit confirmation. If the user says go:

- Stage the exact files the implementation touched — never `git add -A`, which sweeps unrelated worktree files.
- Follow the repo's commit conventions (check `git log` for the prevailing style).
- Reference the issue so GitHub links and auto-closes it: put `Fixes #<NUMBER>` (or `Refs #<NUMBER>` for partial work) in the PR body.

The worktree stays in place either way — the user decides at session end whether to keep or remove it.

## Done when

The issue was fetched and understood (comments included) before anything was created; the work sits on its own branch in a worktree named after the issue; the user was given a chance to add context beyond the issue itself before planning began; a plan citing the issue was approved before the first file edit; the implementation matches that plan with project checks passing; and nothing was committed, pushed, or opened as a PR without the user's explicit go-ahead.
