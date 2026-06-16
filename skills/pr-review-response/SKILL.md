---
name: pr-review-response
description: Respond to code-review comments on a GitHub PR end to end — fetch the unresolved threads, triage real reviewers from CI/bot noise, fix or push back with reasoning, then reply to each thread referencing the fix commit and resolve it. Use when a PR has review comments to work through (a "review round"), the user says reviewers commented / the bot left notes / "address the review", or asks to resolve review conversations. Uses the gh CLI + GitHub GraphQL.
---

# Responding to PR review comments

Work a review round to completion: every actionable comment ends as either a pushed fix or an explained decision, with a reply on the thread and the thread resolved. Reply text is for the reviewer, not the user — so it states what changed and why, and cites the commit.

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
    reviewThreads(first: 50) { nodes {
      id
      isResolved
      comments(first: 5) { nodes { databaseId author { login } path line body } }
    } }
  } } }' \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[]
        | select(.isResolved == false)
        | {thread: .id, comments: [.comments.nodes[] | {id: .databaseId, author: .author.login, path, line, body}]}'
```

- `thread` id looks like `PRRT_...` → used to **resolve**.
- comment `id` is the numeric `databaseId` → used to **reply**.

## 3. Triage

- **Ignore noise.** CI/status bots post reviews with empty `body` and no inline threads. They are not actionable — say so, don't chase them.
- **Real comments** come with a `path`/`line` and substantive `body` (e.g. `copilot-pull-request-reviewer`, humans).
- A second review round only surfaces _new_ unresolved threads; already-resolved ones won't reappear in the filter above.

## 4. Reason about each comment

Judge on the merits — mild criticism both ways, don't rubber-stamp:

- **Agree** → fix it.
- **Right problem, wrong fix** → implement a better fix and explain why you diverged. (For example, when a reviewer points at a symptom, address the underlying cause if the suggested patch would churn a stable interface or its tests.)
- **Disagree** → push back with a concrete reason; don't change code just to silence the bot. Partial fixes are fine when only part of the suggestion holds up on the merits.

## 5. Verify, commit, push

Run the project's checks before committing — consult the repo's `CLAUDE.md`, `README`, or package scripts to find the right commands (typical examples: test runner, linter, formatter, type-checker). Commit on the existing feature branch — never the default branch — and push so the PR head advances. If the repo requires a `Co-Authored-By` trailer or other commit-message convention, follow it. Capture the short SHA for the replies:

```bash
SHA=$(git rev-parse --short HEAD)
```

## 6. Reply to each thread, then conditionally resolve

Reply on every actionable thread (note the `/replies` sub-resource keyed by the comment `databaseId`), citing the commit. **Every reply body must end with the trailer ` | by :robot_face: agent`** so reviewers can tell at a glance the reply came from an agent, not a human. One reply per thread:

```bash
gh api repos/$OWNER/$REPO/pulls/$PR/comments/<COMMENT_DATABASE_ID>/replies \
  -f body="Agreed — <what changed>. Fixed in $SHA. | by :robot_face: agent" --jq '.html_url'
```

Then resolve the thread **only if its first comment was authored by `copilot-pull-request-reviewer`** (the author you captured in step 2). For human reviewers, reply and leave the thread open — they resolve it themselves once they're satisfied with the response. Auto-resolving a human's thread reads as dismissive.

```bash
# Run ONLY when the thread's first-comment author == "copilot-pull-request-reviewer"
gh api graphql -f query='mutation($id: ID!){ resolveReviewThread(input:{threadId:$id}){ thread{ id isResolved } } }' \
  -f id="<PRRT_THREAD_ID>" --jq '.data.resolveReviewThread.thread | "\(.id) resolved=\(.isResolved)"'
```

For a divergent fix or a disagreement on a Copilot thread, the reply must carry the reasoning before you resolve.

## Order & idempotency

Reply _before_ resolving (a resolved thread is easy to overlook). Re-running the step-1 GraphQL query after a round shows what's still `isResolved == false` — a clean way to confirm nothing was missed.

## Done when

Every actionable thread has a fix-or-rationale reply ending with the ` | by :robot_face: agent` trailer; Copilot threads are resolved; human-reviewer threads remain open for the reviewer; fixes are pushed and green; CI/bot empty reviews are acknowledged as non-actionable, not resolved-for-show.
