---
name: pr-review-response
description: Respond to code-review comments on a GitHub PR end to end — fetch the unresolved threads, triage real reviewers from CI/bot noise, fix or push back with reasoning, then reply to each thread referencing the fix commit and resolve it. Use when a PR has review comments to work through (a "review round"), the user says reviewers commented / the bot left notes / "address the review", or asks to resolve review conversations. Uses the gh CLI + GitHub GraphQL.
---

# Responding to PR review comments

Work a review round to completion: every actionable comment ends as either a pushed fix or an explained decision, with a reply on every thread — and resolution left to whoever's actually waiting on it: the human reviewer, or the bot-only cleanup rule in step 8. Reply text is for the reviewer, not the user — so it states what changed and why, and cites the commit.

This skill separates confirmation from execution. Pushback is pressure-tested thread-by-thread while still planning (step 5) — that's the one place per-item back-and-forth belongs. Once the plan is written, a single `ExitPlanMode` approval gates the entire execution phase (steps 7-9): no further per-action confirmations. Plan mode is only entered once there's confirmed work to plan — an empty or all-noise review round closes out in step 3 without ever prompting for it.

## 1. Discover the PR from the current branch

Resolve `OWNER`, `REPO`, and `PR` from the working repo — don't ask the user to supply them. Run this from inside the git repo the user is working on:

```bash
BRANCH=$(git rev-parse --abbrev-ref HEAD)
read OWNER REPO <<<"$(gh repo view --json owner,name --jq '"\(.owner.login) \(.name)"')"
gh pr list --repo "$OWNER/$REPO" --head "$BRANCH" --state open \
  --json number,title,headRefName,url,author \
  --jq '.[] | "\(.number)\t\(.title)\t@\(.author.login)\t\(.url)"'
```

Decide based on the row count:

- **0 rows** → stop. Tell the user no open PR is associated with `$BRANCH` on `$OWNER/$REPO` and do not proceed. Don't push a new PR or switch branches on their behalf.
- **1 row** → set `PR` to that number and continue.
- **2+ rows** → ask the user which PR to work on using AskUserQuestion, listing each PR (number, title, author, URL) as an option. Do not guess.

Once `PR` is fixed:

```bash
export OWNER REPO PR
```

## 2. Fetch reviews and unresolved threads

List reviews to see who reviewed, on which commit, and spot empty bodies:

```bash
gh api repos/$OWNER/$REPO/pulls/$PR/reviews \
  --jq '.[] | "\(.user.login)\t\(.state)\tcommit:\(.commit_id[0:7])\tbody:\(.body[0:40])"'
```

Get unresolved threads with the thread node id (to resolve) **and** each comment's `databaseId` (to reply) plus path/line/body:

```bash
gh api graphql -f query='
{ repository(owner: "'"$OWNER"'", name: "'"$REPO"'") {
  pullRequest(number: '"$PR"') {
    reviewThreads(first: 100) { nodes {
      id
      isResolved
      isOutdated
      comments(first: 20) { nodes { databaseId author { login } path line originalLine body } }
    } }
  } } }' \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[]
        | select(.isResolved == false)
        | {thread: .id, outdated: .isOutdated, comments: [.comments.nodes[] | {id: .databaseId, author: .author.login, path, line, originalLine, body}]}'
```

- `thread` id looks like `PRRT_...` → used to **resolve**.
- comment `id` is the numeric `databaseId` → used to **reply**.
- `outdated: true` → the diff line moved or changed since the comment; the cited `line` is stale (`originalLine` is where it was). Locate the current code by content before fixing — don't trust the line number.
- **Mind the caps.** `reviewThreads(first: 100)` and `comments(first: 20)` are limits, not guarantees. On a busy PR (more threads) or a long back-and-forth (more comments), paginate with `after:` cursors rather than working a silently truncated set — and never resolve a thread whose latest comments you didn't fetch.

## 3. Triage

- **Ignore noise.** CI/status bots post reviews with empty `body` and no inline threads. They are not actionable — say so, don't chase them.
- **Real comments** come with a `path`/`line` and substantive `body` (e.g. `copilot-pull-request-reviewer`, humans).
- A second review round only surfaces _new_ unresolved threads; already-resolved ones won't reappear in the filter above.

**Nothing actionable?** If every unresolved thread is noise (empty-body bot reviews, no substantive inline comments) or there are no unresolved threads at all, stop here. Tell the user the PR has nothing to work through and do not enter plan mode — there's nothing to plan, so asking to plan is just friction.

## 4. Enter plan mode

Now that at least one actionable thread is confirmed, call `EnterPlanMode`. Steps 5-6 below (reasoning about each comment, writing the plan) happen inside plan mode; nothing outside it — no fix, commit, push, reply, or resolve — happens until the plan is approved.

## 5. Reason about each comment

Judge on the merits — mild criticism both ways, don't rubber-stamp:

- **Agree** → fix it.
- **Right problem, wrong fix** → implement a better fix and explain why you diverged. (For example, when a reviewer points at a symptom, address the underlying cause if the suggested patch would churn a stable interface or its tests.)
- **Disagree** → push back with a concrete reason; don't change code just to silence the bot. Partial fixes are fine when only part of the suggestion holds up on the merits.

**Pressure-test anything short of a straight fix, while still planning.** For every diverge or disagree decision, surface the draft reasoning to the user with AskUserQuestion _before_ it goes into the plan — a divergent fix or a "won't fix" is outward-facing, lands on the reviewer's thread under the user's name, and is awkward to retract once posted. Doing this now, one thread at a time, catches objections while they're cheap to fold in; it also means the plan you present next is already vetted, so `ExitPlanMode` is normally a single confirmation rather than a round of revisions. Plain agreement doesn't need this — reserve it for cases where you're diverging from or rejecting the reviewer's suggestion.

Record each decision (fix / diverge / push back) and its reasoning as you go — this is the raw material for the plan in the next step.

## 6. Write the plan and request approval

Write the plan file: for every actionable thread, list the comment, the decision (fix / diverge / push back), the reasoning already discussed with the user for any diverge or push-back decision, and — for fixes — what will actually change. End it with a line naming which human reviewers will be re-requested once execution finishes (step 9) — that's a notification landing in someone else's inbox, so it belongs in what's being approved, not something step 9 does silently after the fact.

Then call `ExitPlanMode` to request approval, passing `allowedPrompts` for every category of mutating action steps 7-9 will run — push commits, post PR replies, resolve review threads, re-request reviewers — so the one approval actually covers the whole execution phase instead of re-prompting at each `git push`/`gh api` call. Do not fix code, commit, push, reply, or resolve anything before the user approves. If the user asks for changes, revise the plan and call `ExitPlanMode` again.

## 7. Verify, commit, push

Skip this step entirely if the plan has zero accepted fixes (an all-diverge/all-pushback round) — there's nothing to commit, and citing a SHA in a reply where no code changed would misrepresent what happened.

Otherwise, before committing, sanity-check the actual diff against what step 6's plan described. If it matches the described intent, proceed. If it diverges in a way that changes what the reviewer or user would see — different files touched, a materially different approach — stop and confirm with the user before committing or pushing. Approval covered the described intent, not an open-ended license to implement anything filed under it.

Run the project's checks before committing — consult the repo's `CLAUDE.md`, `README`, or package scripts to find the right commands (typical examples: test runner, linter, formatter, type-checker). Commit on the existing feature branch — never the default branch — and push so the PR head advances. If the repo requires a `Co-Authored-By` trailer or other commit-message convention, follow it.

If checks fail or the push is rejected (e.g. the remote moved), stop and report the failure to the user — don't force-push, skip a failing check, or silently retry. Fix the root cause and re-run this step, or if that changes the plan meaningfully, go back through step 6 first.

Capture the short SHA for the replies:

```bash
SHA=$(git rev-parse --short HEAD)
```

## 8. Reply to each thread, then conditionally resolve

**Re-check before acting.** Real time has passed since the step-2 snapshot — plan approval and step 7's verify/commit/push can each take a while, and a reviewer may have posted, edited, or (un)resolved something in the meantime. Re-run the step-2 query, or at least re-check the threads you're about to touch, before replying or resolving. If anything material changed — a new comment, a thread that's now resolved, an author list that now includes a human where it didn't — stop and flag it to the user instead of executing the plan as written against stale data.

Reply on every actionable thread (note the `/replies` sub-resource keyed by the comment `databaseId`), citing the commit. **Every reply body must end with the trailer ` | by :robot: agent`** so reviewers can tell at a glance the reply came from an agent, not a human. One reply per thread:

```bash
gh api repos/$OWNER/$REPO/pulls/$PR/comments/<COMMENT_DATABASE_ID>/replies \
  -f body="Agreed — <what changed>. Fixed in $SHA. | by :robot: agent" --jq '.html_url'
```

Then resolve the thread **only if no human has participated in it** — i.e. _every_ comment's author is unambiguously a bot: the login is `copilot-pull-request-reviewer` or ends in `[bot]` (GitHub's convention for bot/App accounts, e.g. `dependabot[bot]`). Treat any account you don't recognize as human — fail safe and leave the thread open rather than guess at "another known bot." Check the full `author` list you captured in step 2, not just the first comment: a human follow-up anywhere in a Copilot-started thread means a person is now in the conversation. Whenever a human has commented, reply and leave the thread open — they resolve it themselves once satisfied. Auto-resolving a human's thread reads as dismissive.

```bash
# Run ONLY when every comment in the thread is bot-authored (no human participant)
gh api graphql -f query='mutation($id: ID!){ resolveReviewThread(input:{threadId:$id}){ thread{ id isResolved } } }' \
  -f id="<PRRT_THREAD_ID>" --jq '.data.resolveReviewThread.thread | "\(.id) resolved=\(.isResolved)"'
```

If the decision on a bot-only thread was to diverge or push back rather than fix outright, make sure that reasoning actually made it into the reply above before running the resolve mutation. This is a content check, not a separate resolution rule — whether to resolve is still gated solely by "no human participant" (above), never by how the disagreement was decided. A divergent fix or disagreement on a thread with a human in it is never resolved by you, regardless of how well-reasoned the reply is.

## 9. Re-request review from human reviewers

Human threads were left open on purpose — but a reviewer won't necessarily notice the new push. Once fixes are pushed and replies posted, re-request review from each human reviewer who left comments, so the ball is visibly back in their court:

```bash
gh api repos/$OWNER/$REPO/pulls/$PR/requested_reviewers -X POST -f 'reviewers[]=<LOGIN>'
```

Skip bots (their threads you already resolved) and skip the PR author. If re-requesting fails (e.g. the reviewer isn't a collaborator), note it rather than forcing it.

## Order & idempotency

Reply _before_ resolving (a resolved thread is easy to overlook). Re-running the step-2 GraphQL query after a round shows what's still `isResolved == false` — a clean way to confirm nothing was missed.

## Done when

Every diverge or push-back decision was pressure-tested and the plan was approved via `ExitPlanMode` before any execution; every actionable thread has a fix-or-rationale reply ending with the ` | by :robot: agent` trailer; threads with no human participant are resolved; threads a human took part in remain open with the reviewer re-requested; fixes are pushed and green; CI/bot empty reviews are acknowledged as non-actionable, not resolved-for-show.
