---
description: Verify the findings of the latest /code-review in this session — one agent checks each finding against the reviewed code, the PR body, the ADRs and CLAUDE.md files and, on a stacked PR, the slices above, by the same rules /review-stack uses — then print each as confirmed, plausible or rejected. Read-only.
---

# Verify Findings

Check whether the findings of a `/code-review` hold up. `/code-review` takes
no context beyond its target, so it cannot know what the PR body, the ADRs or
the slices above already settle. This command hands its findings to one
verifier agent and prints the verdicts. It never edits, commits, pushes or
posts.

## The findings

Take the most recent `/code-review` in this conversation: the target it
reviewed, and its findings — `file`, `line`, `summary` and `failure_scenario`
of each, most severe first. If there is no review in this conversation, or it
found nothing, stop and say so.

## The target

Resolve `OWNER/REPO` once: `gh repo view --json nameWithOwner -q .nameWithOwner`.

- **The review was given a PR** (number or URL). Read
  `gh pr view <N> --json title,baseRefName,headRefName,headRefOid` — the
  head is `<sha>` — and
  `gh api repos/OWNER/REPO/pulls/<N> --jq .stack`. Fetch the head so its code
  can be read without a checkout: `git fetch origin pull/<N>/head`.
  - `null` or `size == 1` → the **PR** row below.
  - otherwise → the **Stacked PR** row. Read the stack:
    `gh api repos/OWNER/REPO/stacks/<.stack.number> --jq '.pull_requests[] | {number, state, head: .head.ref, sha: .head.sha}'`.
    Record `<N>`'s position in the list as `P/SIZE`. The slices above are the
    open PRs after it; the top is the last open PR, and `<top sha>` its head.
    Fetch it too: `git fetch origin pull/<top>/head`.
- **Anything else** (the current diff, a branch, a path) → the **Local** row.

## Verifier prompt

`/plan-lifecycle:review-stack` launches its verifiers with this prompt too.
Fill the placeholders from the target's row:

- **Local** — `<TARGET>`: the changes on `<branch>`. `<CODE>`: the working
  tree. `<DIFF>`: the command that shows the diff the review covered, e.g.
  `git diff HEAD`. `<DECISIONS>`: the repo's ADRs (e.g. `docs/adr/`) and
  CLAUDE.md files in the working tree, and the body of the branch's open PR if
  it has one (`gh pr view --json body`).
- **PR** — `<TARGET>`: PR #<N>. `<CODE>`: `git show <sha>:<path>`. `<DIFF>`:
  `gh pr diff <N>`. `<DECISIONS>`: the PR body (`gh pr view <N> --json body`),
  and the repo's ADRs (e.g. `docs/adr/`) and CLAUDE.md files at `<sha>`.
- **Stacked PR** — as **PR**, except `<TARGET>`: PR #<N> (slice <P>/<SIZE> of
  stack #<S>), and the ADRs and CLAUDE.md files are read at `<top sha>`.

Lines marked `(stack)` go in only for a stacked PR, without the mark.

```
Verify the findings of an automated review of <TARGET> in <OWNER/REPO>. Do
not edit files, switch branches or post anything.

- The reviewed code: <CODE>. Its diff: <DIFF>.
- Documented decisions: <DECISIONS>.
- (stack) The top of the stack: `git show <top sha>:<path>`. The slices above
  this one: <#M (<head>), …, or "none">.

For each finding, in order, give a verdict:
- CONFIRMED: the failure scenario follows from the reviewed code. Quote the
  line.
- PLAUSIBLE: the mechanism is real but the trigger is uncertain (timing,
  environment, config). Say what would confirm it. A realistic race, a rare
  but reachable nil or a partial failure is PLAUSIBLE, not REJECTED.
- REJECTED: only when one of these holds, and cite it:
  - the code doesn't do what the finding says (quote the line);
  - it is guarded in the reviewed code (cite the guard);
  - the PR body, an ADR or a CLAUDE.md records it as a decision (quote it);
  - (stack) a slice above already fixes it or rules it out ("fixed in #<M>",
    quote the line at the top).
and a kind: `bug` (wrong behaviour) or `nit` (cleanup, naming, duplication,
style, test tidiness, or a cost with no observable effect).

Return only a JSON array, one object per finding, in the given order:
[{"verdict": "CONFIRMED|PLAUSIBLE|REJECTED", "kind": "bug|nit", "evidence": "<one or two sentences with the quote>"}]

Findings:
<the review's findings as a JSON array: file, line, summary, failure_scenario>
```

## Verify

Launch one `general-purpose` agent through the `Agent` tool with the filled-in
prompt and wait for it. If it fails or returns an array of the wrong length,
say the findings are unverified and stop.

## Summary

```
## PR #<N> <title>
<base> ← <head>

| # | Where | Verdict | Finding |
|---|-------|---------|---------|
| 1 | `path/to/file.go:123` | confirmed | <one-line summary> |
| 2 | `path/to/file.go:140` | plausible, nit | <one-line summary> |

Rejected:
- `path/to/file.go:88` <one-line summary> — <the verifier's reason, e.g. fixed in #M, or the ADR that records it>
```

For a local target the heading is `## Changes on <branch>`, with no
`<base> ← <head>` line. Leave out `Rejected:` when none were. Keep the
review's ranking and its one-line summaries; the full failure scenarios and
evidence stay in the verifier's result, so do not repeat them unless the user
asks.

End with one line: `<X> of <T> findings hold up, <R> rejected by verification`,
counting confirmed and plausible findings in `<X>`.
