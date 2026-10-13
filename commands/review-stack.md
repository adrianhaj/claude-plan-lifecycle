---
description: Review every open PR of a GitHub stack with /code-review — one review per slice, each scoped to that slice's own diff, launched together, then one summary grouped by slice. Accepts a stack number or any PR (an unstacked PR is a one-slice stack); review flags (effort level, --max-findings) pass through, and --comment posts the findings on each PR, marked for /pr-fix.
argument-hint: [stack or PR number — optional, auto-detects from current branch] [/code-review flags, e.g. high --comment]
---

# Review Stack

Run `/code-review` on each open PR of a stack and report the findings slice by
slice. This command reviews only — it never edits, commits or pushes. With
`--comment`, it posts each slice's findings on that slice's PR, marked so
`/plan-lifecycle:pr-fix <stack>` treats them as review findings to verify (not
your own notes) and fixes each in the slice that owns it.

## Resolving the stack

Resolve `OWNER/REPO` once: `gh repo view --json nameWithOwner -q .nameWithOwner`.

Split `$ARGUMENTS`: if the first token is a number it is the target; everything
after it (or all of `$ARGUMENTS` when the first token is not a number) is
`REVIEW_ARGS`, passed verbatim to every review (an effort level,
`--max-findings <n>`). `--comment` is taken out of `REVIEW_ARGS` and sets
`POST`: the reviews do not post, Step 2 does. `--fix` stops the command: each
review would apply its findings to the same checkout. Tell the user to post
with `--comment` and fix with `/plan-lifecycle:pr-fix` instead.

- **The target is a bare number** — try it as a **stack number** first:
  `gh api repos/OWNER/REPO/stacks/<N>`. Stacks draw from the same number
  sequence as issues and PRs, so a number is a stack *or* a PR, never both.
  A 200 means `STACK_NUM = N`. A 404 means it is a PR number; continue below.
  Any other status: stop and show the error — do not guess.
- **PR number** (given, or auto-detected via `gh pr view --json number` for the
  current branch). If there is no PR, stop and tell the user. Read
  `gh api repos/OWNER/REPO/pulls/<PR> --jq .stack`:
  - `null` or `size == 1` → the PR is its own one-slice stack: `TARGET_PRS` is
    that PR alone, `SIZE = 1`, with its `title`, `head.ref`, `base.ref` and
    head `sha` from `gh pr view <PR> --json title,headRefName,baseRefName,headRefOid`.
    Wherever `slice <P>/<SIZE> of stack #<S>` or `Stack #<S>` appears below,
    write `PR #<N>` instead. Skip to the pre-flight.
  - otherwise `STACK_NUM = .stack.number`.

Fetch the stack once, with everything the summary needs:
`gh api repos/OWNER/REPO/stacks/<STACK_NUM> --jq '.pull_requests[] | {number, state, title, head: .head.ref, base: .base.ref, sha: .head.sha}'`.
`TARGET_PRS` = every PR with `state == open`, in stack order (bottom first);
merged slices are skipped. Record each one's position in the full list as
`P/SIZE`. If none is open, stop.

**Pre-flight:** `git status --porcelain` must have no non-`??` lines. A review
can pull uncommitted changes into its scope, which would put them in all `<K>`
reviews.

Print the header: `Stack #<S> — reviewing <K> open PRs: #<a>, #<b>, …` (or
`PR #<N> (not stacked)`).

## Step 1 — Launch one review per slice

In a single message, invoke the `code-review` skill through the `Skill` tool
once per PR in `TARGET_PRS`, with arguments
`REVIEW_ARGS https://github.com/OWNER/REPO/pull/<N>` — flags first: an effort
level placed after the target is not read, and the review silently reuses the
last level instead. Pass the PR URL, not the bare number, so each review
resolves the repository regardless of the working directory.

A review of a PR diffs it against its base branch, which in a stack is the
slice below it — so each review's findings are about that slice's changes. The
surrounding code a review reads for context (callers, enclosing functions) can
come from the local checkout, which is one slice at most: a finding that rests
on code outside the diff may describe a later slice's version of it. Do not
review the code yourself and do not launch a second review of a PR whose review
is still running.

Each `Skill` call either returns its review's result at once ("completed
(forked execution). Result: …") or starts the review as a background agent
("launched … running in the background") that reports with a task notification
when it finishes — one run can get both. A review is done when its result is
in, whichever way it came; wait until all `<K>` are done before Step 2. If one
fails, report it as failed in its slice's block rather than retrying.

Each result ends with the review's findings as a JSON array — `file`, `line`,
`summary`, `failure_scenario`, most severe first. Steps 2 and 3 read them from
there.

## Step 2 — Post the findings (`POST` only)

Posting here, not in the reviews, is what lets every comment carry the marker
`<!-- plan-lifecycle:review-stack -->` that `/pr-fix` keys on. For each slice
with findings, post one review:

- **Where each finding goes.** In `gh pr diff <N>`, a hunk header
  `@@ -a,b +c,d @@` covers lines `c` to `c+d-1` of the new file (no `,d`
  means one line; `,0` covers none).
  - `file:line` inside a hunk → an inline comment in the review.
  - `file` in the diff, line outside every hunk → a file-level comment, posted
    after the review. GitHub rejects a whole review if one of its line
    comments is off the diff, and the review call can't carry file-level
    comments; posted on its own, one is a thread that can be resolved like
    any other.
  - `file` not in the diff → the review body.
- **Post the review** with `gh api repos/OWNER/REPO/pulls/<N>/reviews --method POST --input <file>`:

  ```json
  {
    "commit_id": "<the slice's sha>",
    "event": "COMMENT",
    "body": "Automated review (`/review-stack <REVIEW_ARGS>`), slice <P>/<SIZE> of stack #<S>: <n> findings.\n\n<one entry per finding whose file is not in the diff: **`path:line`** summary, then failure_scenario>\n\n<!-- plan-lifecycle:review-stack -->",
    "comments": [
      { "path": "<file>", "line": <line>, "side": "RIGHT",
        "body": "**<summary>**\n\n<failure_scenario>\n\n<!-- plan-lifecycle:review-stack -->" }
    ]
  }
  ```

- **Post each file-level comment** with `gh api repos/OWNER/REPO/pulls/<N>/comments --method POST -f commit_id=<the slice's sha> -f path=<file> -f subject_type=file -f body=<body>`,
  the body being `` **`path:line`** <summary> ``, then the failure scenario
  and the marker, each separated by a blank line.

`commit_id` is the head recorded before the reviews ran. Before posting a
slice, check `gh pr view <N> --json headRefOid` still matches it; if the
branch moved, the review and the diff no longer describe the same code, so do
not post that slice — say so in its block.

Post each slice once. If a post fails, show the error in that slice's block
and carry on with the others.

## Step 3 — Summary

One block per slice, bottom-up:

```
## Slice <P>/<SIZE> — PR #<N> <title>
<base> ← <head>
Posted: <the review's html_url>        (POST only)

| # | Where | Finding |
|---|-------|---------|
| 1 | `path/to/file.go:123` | <one-line summary> |
```

`No findings.` for a clean slice. Keep each review's own ranking (most severe
first) and its one-line summaries; the full failure scenarios stay in the
review results, so do not repeat them unless the user asks.

End with one line: `Stack #<S>: <X> findings across <K> PRs (#<a>: <n>, #<b>: <m>, …)`.

With `POST`, add: "Each comment carries the review-stack marker, so
`/plan-lifecycle:pr-fix <S>` verifies each finding like a reviewer's comment
rather than treating it as your own note."

When `REVIEW_ARGS` set an effort level or `--max-findings`, add: "`/code-review`
keeps <those settings> for later runs too; type another level, or
`--max-findings default`, to change them."
