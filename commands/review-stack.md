---
description: Review every open PR of a GitHub stack with /code-review — one review per slice, each scoped to that slice's own diff, launched together, then one summary grouped by slice. Accepts a stack number or any PR in the stack; review flags (effort level, --comment) pass through.
argument-hint: [stack or PR number — optional, auto-detects from current branch] [/code-review flags, e.g. high --comment]
---

# Review Stack

Run `/code-review` on each open PR of a stack and report the findings slice by
slice. This command reviews only — it never edits, commits or pushes. With
`--comment`, each review posts its findings on its own PR, and
`/plan-lifecycle:pr-fix <stack>` then fixes each one in the slice that owns it.

## Resolving the stack

Resolve `OWNER/REPO` once: `gh repo view --json nameWithOwner -q .nameWithOwner`.

Split `$ARGUMENTS`: if the first token is a number it is the target; everything
after it (or all of `$ARGUMENTS` when the first token is not a number) is
`REVIEW_ARGS`, passed verbatim to every review (an effort level, `--comment`,
`--max-findings <n>`).

- **The target is a bare number** — try it as a **stack number** first:
  `gh api repos/OWNER/REPO/stacks/<N>`. Stacks draw from the same number
  sequence as issues and PRs, so a number is a stack *or* a PR, never both.
  A 200 means `STACK_NUM = N`. A 404 means it is a PR number; continue below.
- **PR number** (given, or auto-detected via `gh pr view --json number` for the
  current branch). If there is no PR, stop and tell the user. Read
  `gh api repos/OWNER/REPO/pulls/<PR> --jq .stack`:
  - `null` or `size == 1` → stop: "PR #<N> is not part of a stack — run
    `/code-review <N>` instead."
  - otherwise `STACK_NUM = .stack.number`.

Fetch `repos/OWNER/REPO/stacks/<STACK_NUM>`. `TARGET_PRS` = every PR in
`.pull_requests[]` with `state == open`, in stack order (bottom first); merged
slices are skipped. Record each one's position in the full list as `P/SIZE`. If
none is open, stop.

Print the header: `Stack #<S> — reviewing <K> open PRs: #<a>, #<b>, …`.

## Step 1 — Launch one review per slice

In a single message, invoke the `code-review` skill through the `Skill` tool
once per PR in `TARGET_PRS`, with arguments
`https://github.com/OWNER/REPO/pull/<N> REVIEW_ARGS`. Pass the PR URL, not the
bare number, so each review resolves the repository regardless of the working
directory.

A review of a PR diffs it against its base branch, which in a stack is the
slice below it — so each review covers exactly that slice's changes, and no
branch needs checking out. Do not review the code yourself and do not launch a
second review of a PR whose review is still running.

The reviews run as background agents and each reports when it finishes. Wait
until all `<K>` have reported before Step 2; if one fails, report it as failed
in its slice's block rather than retrying.

## Step 2 — Summary

One block per slice, bottom-up:

```
## Slice <P>/<SIZE> — PR #<N> <title>
<base> ← <head>

| # | Where | Finding |
|---|-------|---------|
| 1 | `path/to/file.go:123` | <one-line summary> |
```

`No findings.` for a clean slice. Keep each review's own ranking (most severe
first) and its one-line summaries; the full failure scenarios stay in the
review results, so do not repeat them unless the user asks.

End with one line: `Stack #<S>: <X> findings across <K> PRs (#<a>: <n>, #<b>: <m>, …)`.
When `REVIEW_ARGS` contained `--comment`, add: "Findings are posted on each PR —
run `/plan-lifecycle:pr-fix <S>` to fix them slice by slice."
