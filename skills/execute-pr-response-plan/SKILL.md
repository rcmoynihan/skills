---
name: execute-pr-response-plan
description: Execute a response plan produced by plan-pr-comment-responses — implement every comment's TODOs, run the project's tests/validation until green, have codex:rescue verify the work matches the plan with no scope creep, commit + push, drive PR CI to green, then post the drafted replies and resolve every thread except push-backs / significant divergences. Use when the user wants to carry out / apply / ship an already-written PR response plan.
argument-hint: "path to the response-plan .md (optional; defaults to the newest pr-*-response-plan.md in the plugin's temp dir)"
disable-model-invocation: true
---

# Execute a PR response plan

You are the PR author's assistant, carrying a **response plan** produced by the
`plan-pr-comment-responses` skill all the way to done: implement every TODO, get validation green,
have Codex independently verify the work, commit + push, drive CI to green, and post the drafted
replies — resolving every thread except the ones where the plan pushed back or significantly
diverged.

Unlike the planning skill, this one **edits code, runs tests, commits, pushes, and posts to the
PR.** It runs the whole pipeline **autonomously**, including the irreversible outward actions.

**Honor the global rules throughout:**
- No change-narration in code, comments, docs, or PR replies — describe the resulting state, not
  what changed or your process. (Commit messages describing the diff are fine.)
- Commit messages end with the `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`
  trailer.
- Work on the PR's head branch; never commit on the default branch.

## Step 1 — Load the plan

Resolve the plan file: use the path argument if given, otherwise pick the newest match of
`"${TMPDIR:-/tmp}"/code-goblin-pro/pr-*-response-plan.md`:

```bash
ls -t "${TMPDIR:-/tmp}"/code-goblin-pro/pr-*-response-plan.md 2>/dev/null | head -1
```

Read it and extract:
- The **PR ref** (the plan embeds the PR number / URL / title).
- Every `C#` item: its **decision** (implement / alternate / push back / need-info / defer), its
  **drafted PR reply**, and its **TODO block** (files/functions, steps, done-when).
- The **tests + verification** section and any specific test cases it names.
- Any **cross-cutting note** the plan flagged (section 6) — posted top-level only if it genuinely
  belongs on no single comment thread; the plan omits it by default.

Build a working table keyed by comment: `comment_id → {decision, todo, reply, status}`. You'll fill
in `status` as you implement. If no plan file is found, stop and ask for the path.

## Step 2 — Ready the repo

Confirm you're in the PR's repo and on the PR **head branch**:

```bash
gh pr view <ref> --json number,headRefName,headRepository,url,title
git switch <headRefName>     # if not already on it; never work on the default branch
```

Identify the project's test / lint / build commands by reading the repo's `CLAUDE.md`,
`package.json`, `Makefile`, `pyproject.toml`, and CI config. **Reuse the commands that already
exist** rather than inventing your own.

## Step 3 — Implement every TODO

Work the `C#` blocks in the plan's sequencing order. For each, make exactly the changes its TODO
specifies and record an outcome `status`:

- **implemented-as-planned** — done as the TODO outlined.
- **implemented-alternate** — done via the plan's stated alternate approach.
- **no-code** — duplicate of another C#; satisfied by that primary fix (note which).
- **pushed-back** — plan decided to push back; no code change.
- **diverged** — you had to deviate materially from the drafted fix (note why).
- **deferred** — plan defers it to a tracked follow-up; no code change now.

Stay inside the plan's scope. The plan's "what not to change" notes are binding — do not opportun­
istically refactor or fix unrelated things.

## Step 4 — Local validation

Run the project's tests, lint, and build (the commands from Step 2, plus any cases the plan's
verification section calls out). Fix failures and re-run until green. Keep the final passing output
to report later. **Do not proceed to commit on red local validation.**

## Step 5 — Independent verification via `codex:rescue`

Invoke the **`codex:rescue`** skill (via the Skill tool). Hand it the plan file path **and** the
working-tree diff (`git diff`), and ask it to verify:

1. **Coverage** — every `C#` item is addressed exactly as the plan outlined (or correctly recorded
   as no-code / pushed-back / deferred).
2. **No scope creep** — nothing in the diff that the plan didn't call for.
3. **Nothing unexpected** — no stray debug code, unrelated edits, commented-out blocks, or secrets.

Treat its findings as a punch list: fix genuine gaps, revert scope creep, and correct or justify
flagged divergences. Keep your approach where its critique isn't convincing — you have the final
say. If you change anything, re-run Step 4. If `codex:rescue` reports Codex is unavailable, note
that and proceed.

## Step 6 — Commit & push

Commit the work with a clear message describing the change, ending with the required
`Co-Authored-By` trailer, then push to the PR head branch:

```bash
git add -A && git commit -m "<summary>

<details>

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
git push
```

## Step 7 — Drive CI to green

Watch the PR's checks:

```bash
gh pr checks <ref> --watch
```

On failure, diagnose from the failing jobs (`gh pr checks <ref>`, `gh run view --log-failed`), fix
the cause, re-run local validation (Step 4), commit, push, and re-watch. **Cap at ~3 rounds.** If
CI is still red after that, stop and hand the user the failing job details and everything you
tried — do not post replies on a red PR. Proceed to Step 8 only once CI is green.

## Step 8 — Post replies & resolve threads

Only after CI is green. For each `C#`, post its **drafted reply** in-thread:

- **Inline review comment** — reply to the originating comment:
  ```bash
  gh api --method POST \
    "repos/{owner}/{repo}/pulls/{number}/comments/{comment_id}/replies" \
    -f body='<drafted reply>'
  ```
- **General PR comment** (the reviewer's comment was itself a non-inline PR comment) — reply with
  `gh pr comment <ref> --body '<drafted reply>'`.

Then decide resolution by outcome:

- **Resolve the thread** for settled outcomes — **implemented-as-planned**,
  **implemented-alternate**, and **no-code** duplicates satisfied by their primary.
- **Leave unresolved** (reply only) for **pushed-back**, **diverged**, **need-info**, and
  **deferred** — the reviewer should see and engage. The reply explains the pushback / divergence /
  follow-up.

Resolving a thread is **GraphQL-only** (REST can't resolve). Fetch thread node IDs, map each
thread to a `C#` by its first comment's `databaseId`, then resolve the settled ones:

```bash
gh api graphql -f query='query($o:String!,$r:String!,$n:Int!){repository(owner:$o,name:$r){
  pullRequest(number:$n){reviewThreads(first:100){nodes{id isResolved
  comments(first:1){nodes{databaseId path}}}}}}}' -F o=<owner> -F r=<repo> -F n=<number>

gh api graphql -f query='mutation($id:ID!){resolveReviewThread(input:{threadId:$id}){thread{isResolved}}}' -f id=<threadId>
```

**Reply inline; never summarize.** Every reply is posted in-thread on the exact comment it answers.
Do **not** post a top-level "here's what I did," "summary of changes," or status comment — that noise
is exactly what to avoid. Post a top-level comment **only** when the plan's section 6 flagged a genuine
cross-cutting concern that belongs on no single thread; otherwise post nothing top-level.

## Step 9 — Report back

Return a concise summary to the user:
- Implemented items grouped by outcome status.
- Final local validation + CI status (green).
- Which threads were resolved and which were left open, with the reason for each open one.
- The commit SHA(s) pushed and the PR URL.

## Quality bar

- Every `C#` in the plan ends with a recorded outcome, a posted reply, and a resolve/leave-open
  decision consistent with that outcome — none skipped.
- No scope creep: the pushed diff contains only what the plan called for (Step 5 enforces this).
- Never resolve a pushed-back, diverged, need-info, or deferred thread.
- Never post replies or resolve threads while CI is red.
- Replies are inline, in-thread. Never post a summary or "here's what I did" top-level comment;
  top-level is reserved for a genuine flagged cross-cutting concern (section 6), if any.
- If blocked (no plan file, CI still red after the cap, a TODO that can't be implemented as
  specified), stop and report clearly rather than guessing or forcing it through.
