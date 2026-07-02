---
name: plan-pr-comment-responses
description: Turn a PR's diff and reviewer comments into a concrete, comment-by-comment plan for addressing them — per-comment decision (implement / alternate / push back / defer), implementation TODOs, drafted PR replies, a verification plan, and a re-review checklist. Plans only; does not edit code or post replies. Use when the user wants to work through / triage / plan responses to PR review comments or feedback.
argument-hint: "PR number / URL / branch, or paste the diff + review comments"
---

# Plan PR comment responses

You are acting as the PR author's assistant. Your job is to produce a *concrete, implementable
plan* that another engineer/agent can follow to address every point raised by reviewers, together
with a proposed written reply for each comment (what the author should post in the PR, including
where to push back).

**You plan only.** You do not edit code, push commits, or post replies. The deliverable is a
markdown report written to a file (see [Output](#output)).

## Gather inputs

You need two things: the **code changes** (a diff) and the **review feedback** (inline, threaded,
and general comments).

### When given a PR ref (number, URL, or branch) — auto-fetch with `gh`

This is the default path. Resolve the repo and pull the diff plus all comment surfaces:

```bash
gh repo view --json nameWithOwner                                  # owner/repo, if needed
gh pr diff <ref>                                                   # the diff
gh pr view <ref> --json title,body,url,headRefName,baseRefName,files
gh api repos/{owner}/{repo}/pulls/{number}/comments                # inline/threaded review comments
gh api repos/{owner}/{repo}/pulls/{number}/reviews                 # review summary bodies
gh api repos/{owner}/{repo}/issues/{number}/comments               # general PR comments
```

`gh` can usually infer `{owner}/{repo}` and the PR number from the ref or the current checkout;
fall back to `gh repo view --json nameWithOwner` when you need them explicit. Inline review
comments carry `path`, `line`, `body`, `user`, and `in_reply_to_id` — use these for `Location`
and for grouping threads.

### Fallback — pasted content

If no PR ref is available or `gh` can't reach the PR, use the diff and comments the user pasted
(full diff preferred; otherwise file snippets + described intent).

### If inputs are incomplete

If you cannot obtain **either** a usable diff **or** any review comments, stop and tell the user
exactly what's missing. Do not guess your way through a plan without the inputs.

## Objectives

1. Understand what the branch is doing and why.
2. Interpret reviewer feedback precisely:
   - **Literal ask:** what they directly request or question.
   - **Spirit:** the underlying goal (readability, correctness, architecture, perf, tests, safety,
     future-proofing).
3. Decide an approach for each comment: accept and implement / accept with modification / push
   back (with justification) / defer (with explicit follow-up + rationale) / need more info.
4. Produce: an exhaustive implementation plan, a comment-by-comment reply plan, verification steps,
   a risk assessment, and a re-review checklist.

Pay special attention to any comment calling out that these changes introduce a pattern or utility
that **copies existing functionality you could have reused**, or **differs from a direction already
used for similar features** that you should adopt instead of reinventing.

## Required workflow (in order)

### Step 1 — Project familiarization

Identify entry points, key modules and ownership boundaries, style/lint conventions, existing
patterns similar to the changed code, the test strategy and mocking style, and error
handling/logging conventions. Summarize in 5–15 bullets, focused only on what matters for the diff.

### Step 2 — Diff comprehension

Summarize the branch's intent: what behavior changes for users/systems, what internal interfaces
changed, what invariants are assumed, any migration/backward-compat needs. List all changed files
with a one-line purpose each. Flag public API changes, data-model changes, performance-sensitive
paths, concurrency/async implications, and security/privacy implications.

### Step 3 — Review comment parsing

For each comment extract: a stable Comment ID (invent one if none exists, keep it consistent);
location (file + line if provided); reviewer tone if evident (blocker / suggestion / nit); the
literal ask (one sentence); the spirit (one sentence); risk if ignored (low/med/high + brief
reason).

### Step 4 — Decision per comment

Choose one: **implement as requested** / **implement alternate approach** (explain tradeoffs) /
**push back** (why current code is better, or request is out of scope / risky) / **need more info**
(list what's missing and propose a default decision if no reply). If pushing back, propose a
compromise when possible. If the reviewer asks a question, answer it and explain implications.

### Step 5 — Implementation plan

Concrete and actionable. You may group changes by theme for readability, but the TODO section must
be **comment-by-comment**. For each change: exact files/functions to edit; what to add/remove/
rename; any new helpers/utilities; any code movement; complexity estimate (S/M/L) and sequencing
constraints. Call out quick wins vs risky/architectural changes, places that may break existing
behavior, and what *not* to change (avoid scope creep).

**Granularity requirement:** every review comment (every C# item, including duplicates) gets its
**own dedicated TODO block**. Do not batch multiple comment IDs into one block, even when they
touch the same code. For duplicates/overlaps the TODO may be "no new code," but must still verify
the duplicate is satisfied by the primary fix and draft that comment's reply (linking to the
primary comment ID/fix).

### Step 6 — Testing + verification plan

Unit/integration/e2e tests to add or update; test cases mapped to reviewer concerns; how to run
locally and in CI; a benchmark plan if there are perf concerns; regression risks and how tests
mitigate them.

### Step 7 — PR response plan (comment-by-comment)

For every comment draft a reply that acknowledges the concern, states the planned action (or
pushback) clearly, mentions follow-up commit(s) where applicable, and stays professional and
concise. Link duplicate comments together.

### Step 8 — Adversarial review via `codex:rescue`

Before finalizing, invoke the **`codex:rescue`** skill (the `/codex:rescue` skill, via the Skill
tool) to perform an adversarial review of your response plan — especially every comment marked
**push back** or **alternate approach**. Hand it the drafted plan (or its file path) and ask it to
**steelman the reviewers' position**. Integrate any convincing points; where its argument is not
convincing and your original response is better, keep yours. You have the final say.

Run this pass before writing the final file (or update the file afterward) so the on-disk report
reflects the reviewed plan. If `codex:rescue` reports Codex is unavailable, note that in the report
and proceed without the pass rather than failing.

### Step 9 — Final checklist

Provide a "ready to re-request review" checklist: all comments addressed (with links/notes), tests
passing, lint/formatting applied, docs updated if needed, and any follow-up TODOs tracked (ticket/issue
refs if provided). Replies are made inline per comment — do not plan a summary or "here's what I did"
top-level comment; the only top-level message is the optional cross-cutting note in section 6.

## Output

Write the full report to a markdown file in the plugin's dir under the **OS system temp directory** —
explicitly **not** the session scratchpad and **not** the workspace:

```
"${TMPDIR:-/tmp}/code-goblin-pro/pr-<ref>-response-plan.md"
```

Derive `<ref>` from the PR (e.g. its number, or a sanitized branch name). Then return to the user
**only**: the absolute file path plus a 2–3 line summary (PR title, number of comments triaged,
count of push-backs). The file is the deliverable; the chat gets the pointer and summary.

### Required report structure

```markdown
# 1) Project understanding (relevant highlights)
- ...

# 2) What this branch does (diff summary)
- Intent:
- User/system impact:
- Files changed:
  - path/to/file: purpose
- Key design choices observed:

# 3a) Review comments breakdown
## C1 — <short title>
- Location:
- Literal ask:
- Spirit:
- Risk if ignored:
- Decision: implement / alternate / push back / need info / defer
- Proposed PR reply:
- Implementation notes:
(repeat for every comment)

# 3b) Highlighted pushbacks
For each comment you push back on: restate the original comment and your response, for quick
author reference.

# 4) Implementation plan
## 4a) Comment goals table
- C1: <one-line goal>
- C2: <one-line goal>
(repeat for every comment)

## 4b) Comment-by-comment TODOs (one block per comment)
### C1 — <short title>
- Files/Functions:
- Steps:
- Notes/constraints:
- Done when:
(repeat for every C#; never combine comments into one block)

## 4c) Optional thematic rollup (readability only)
- Theme A: C1, C4, C7
(optional; must not replace the comment-by-comment TODOs)

# 5) Tests + verification
- Added/updated tests:
- Commands:
- Edge cases:
- Perf/rollout (if relevant):

# 6) Cross-cutting note (only if needed)
(Leave empty by default. Fill in only if there is a genuine cross-cutting concern that belongs on no
single comment thread. This is NOT a summary or a "here's what changed" comment — the executor replies
inline per comment and posts this top-level note only when it is present.)

# 7) Re-review checklist
- [ ] ...
```

## Quality bar

- Do not skip any comment; every comment maps to a plan item **and** a proposed reply.
- Section 4 must show the explicit **1:1 mapping of comment ID → TODO block**, with a block for
  every C#.
- Do not hand-wave with "refactor" or "clean up" — specify what to do.
- Prefer minimal, safe changes unless the feedback demands a larger redesign.
- If tradeoffs exist, articulate them and choose one; do not leave decisions unresolved.
- If uncertain due to missing context, state assumptions explicitly and propose the safest default.
