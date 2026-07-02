---
name: pr-review-response
description: Respond to code-review comments on a GitHub PR end to end — fetch the unresolved threads, triage real reviewers from CI/bot noise, reason through each one and pressure-test any pushback immediately, implement the agreed fixes locally, then get one go-ahead — scaled to how big and clean the real diff turned out — before pushing, replying to every thread, and resolving the ones no human is waiting on. Use when a PR has review comments to work through (a "review round"), the user says reviewers commented / the bot left notes / "address the review", or asks to resolve review conversations. Uses the gh CLI + GitHub GraphQL.
---

# Responding to PR review comments

Work a review round to completion: every actionable comment ends as either a pushed fix or an explained decision, with a reply on every thread — and resolution left to whoever's actually waiting on it: the human reviewer, or the bot-only cleanup rule in *Reply to each thread, then conditionally resolve*. Reply text is for the reviewer, not the user — so it states what changed and why, and cites the commit.

This skill confirms once, late, and on the real artifact. Pushback is pressure-tested thread-by-thread the moment it's decided (*Reason about each comment*) — that's cheap, catches objections early, and never waits on a diff that doesn't exist yet. Everything else — the fixes, the actual diff, the reply/resolve/re-request plan — gets implemented first, locally and uncommitted, so there's no cost to doing it before anyone signs off, and only then handed to the user in *Confirm before executing*. That confirmation scales to what the diff actually looks like: small and expected gets a compact go/no-go, anything bigger or surprising gets walked through in more detail. Either way, nothing pushes, replies, resolves, or re-requests before it.

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

**Nothing actionable?** Stop here if the GraphQL query from *Fetch reviews and unresolved threads* returns zero unresolved threads, or if every thread it does return fails the "real comment" test above (no substantive `path`/`line`/`body`). Either way, tell the user the PR has nothing to work through and stop — there's nothing to implement or confirm.

## 4. Reason about each comment

Judge on the merits — mild criticism both ways, don't rubber-stamp:

- **Agree** → fix it.
- **Right problem, wrong fix** → implement a better fix and explain why you diverged. (For example, when a reviewer points at a symptom, address the underlying cause if the suggested patch would churn a stable interface or its tests.)
- **Disagree** → push back with a concrete reason; don't change code just to silence the bot. Partial fixes are fine when only part of the suggestion holds up on the merits.

**Pressure-test anything short of a straight fix, right away.** For every diverge or disagree decision, surface the draft reasoning to the user with AskUserQuestion before moving on — a divergent fix or a "won't fix" is outward-facing, lands on the reviewer's thread under the user's name, and is awkward to retract once posted. Doing this now, one thread at a time, catches objections while they're cheap to fold in, and means the diff you implement next never has to be unwound over a disagreement the user would have rejected. Plain agreement doesn't need this — reserve it for cases where you're diverging from or rejecting the reviewer's suggestion.

Record each decision (fix / diverge / push back) and its reasoning as you go — you'll need it both for the reply text and for the summary in *Confirm before executing*.

## 5. Implement and verify

Skip straight to *Confirm before executing* if the round has zero accepted fixes (an all-diverge/all-pushback round) — there's nothing to implement.

Otherwise, implement each fix decision now, on the existing feature branch — never the default branch. Nothing here is committed or pushed yet, so there's no cost to doing this before anyone signs off: it's local and fully reversible.

Run the project's checks — consult the repo's `CLAUDE.md`, `README`, or package scripts to find the right commands (typical examples: test runner, linter, formatter, type-checker).

If checks fail, or implementing a fix reveals the reviewer's suggestion doesn't actually work as described, stop and report it to the user rather than forcing something through — don't skip a failing check or silently reach for a materially different approach than what was reasoned through in the previous step. Fold any real change of approach back into that step's decision before continuing.

## 6. Confirm before executing

This is the one gate before anything touches the shared PR. Nothing commits, pushes, replies, resolves, or re-requests before it — and it always runs, no matter how small the round.

Always include:

- One line per actionable thread: the comment, the decision (fix / diverge / push back), and — for fixes — what actually changed.
- The real diff stat (e.g. `git diff --stat`) for what *Implement and verify* produced — the actual result, not a description of intent.
- Which human reviewers will be re-requested once execution finishes — a notification landing in someone else's inbox belongs in what's being approved, not something *Re-request review* does silently afterward.

Then scale the level of detail to what the diff actually looks like:

- **Small and expected** — as a rule of thumb, 3 files or fewer and roughly 30 changed lines or fewer, matching what *Reason about each comment* predicted — keep it to the summary above and ask one compact go/no-go question with AskUserQuestion.
- **Larger, touching unexpected files, or diverging from what was reasoned through** — walk through it in more detail first (key hunks or a one-line-per-file breakdown), call out explicitly what changed from expectations, then ask.

Either way, get an explicit answer via AskUserQuestion before continuing — don't infer approval from silence or from the user moving on to something else. If the user asks for changes, go back to *Implement and verify* (or *Reason about each comment*, if the objection is about the decision itself, not the code) and return here again before executing.

## 7. Commit and push

Commit on the existing feature branch — never the default branch — and push so the PR head advances. If the repo requires a `Co-Authored-By` trailer or other commit-message convention, follow it.

If committing or pushing fails (e.g. a hook rejects the commit, or the remote moved), stop and report it to the user — don't skip hooks, force-push, or retry blindly. Fix the root cause and re-run this step.

Capture the short SHA for the replies:

```bash
SHA=$(git rev-parse --short HEAD)
```

## 8. Reply to each thread, then conditionally resolve

**Re-check before acting.** Real time has passed since the snapshot fetched in *Fetch reviews and unresolved threads* — implementing, verifying, and waiting on the *Confirm before executing* gate can each take a while, and a reviewer may have posted, edited, or (un)resolved something in the meantime. Re-run that query, or at least re-check the threads you're about to touch, before replying or resolving. If anything material changed — a new comment, a thread that's now resolved, an author list that now includes a human where it didn't — stop and flag it to the user instead of proceeding against stale data.

Reply on every actionable thread (note the `/replies` sub-resource keyed by the comment `databaseId`), citing the commit. **Every reply body must end with the trailer ` | by :robot: agent`** so reviewers can tell at a glance the reply came from an agent, not a human. One reply per thread:

```bash
gh api repos/$OWNER/$REPO/pulls/$PR/comments/<COMMENT_DATABASE_ID>/replies \
  -f body="Agreed — <what changed>. Fixed in $SHA. | by :robot: agent" --jq '.html_url'
```

Then resolve the thread **only if no human has participated in it** — i.e. _every_ comment's author is unambiguously a bot: the login is `copilot-pull-request-reviewer` or ends in `[bot]` (GitHub's convention for bot/App accounts, e.g. `dependabot[bot]`). Treat any account you don't recognize as human — fail safe and leave the thread open rather than guess at "another known bot." Check the full `author` list you captured in *Fetch reviews and unresolved threads*, not just the first comment: a human follow-up anywhere in a Copilot-started thread means a person is now in the conversation. Whenever a human has commented, reply and leave the thread open — they resolve it themselves once satisfied. Auto-resolving a human's thread reads as dismissive.

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

Skip bots — same test as *Reply to each thread, then conditionally resolve* (login is `copilot-pull-request-reviewer` or ends in `[bot]`); their threads you already resolved — and skip the PR author. If re-requesting fails (e.g. the reviewer isn't a collaborator), note it rather than forcing it.

## Order & idempotency

Reply _before_ resolving (a resolved thread is easy to overlook). Re-running the query from *Fetch reviews and unresolved threads* after a round shows what's still `isResolved == false` — a clean way to confirm nothing was missed.

## Done when

Every diverge or push-back decision was pressure-tested before implementation; the diff, replies, and re-review plan were confirmed in *Confirm before executing* before anything was pushed, posted, or resolved; every actionable thread has a fix-or-rationale reply ending with the ` | by :robot: agent` trailer; threads with no human participant are resolved; threads a human took part in remain open with the reviewer re-requested; fixes are pushed and green; CI/bot empty reviews are acknowledged as non-actionable, not resolved-for-show.
