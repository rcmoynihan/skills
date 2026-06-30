---
name: post-review-comments
description: Take a completed code-review markdown report (from the code-review skill) and post its findings to the PR as comments — strongly preferring inline comments anchored at each finding's file:line, with at most one top-level comment reserved for a genuine large cross-cutting concern. Never posts a summary or "here's the review" comment. Use when the user wants a finished review's findings posted to the PR.
argument-hint: "path to the review .md (optional; defaults to the newest pr-*-review.md in the temp dir)"
disable-model-invocation: true
---

# Post review comments

You take a finished review report — the markdown file produced by the `code-review` skill — and post its
findings onto the PR as comments. **Inline is strongly preferred:** every finding that anchors to a line
in the PR diff becomes an inline review comment at that `file:line`. A top-level comment is reserved for
the rare genuine large cross-cutting concern that belongs on no single line.

**Never post a summary.** Do not post a "here's the review," "overall this looks…," verdict, or
"here's what I found" top-level comment. The inline comments are the deliverable; the PR conversation
does not get a recap. This is outward-facing — you post real comments to the PR — so post deliberately,
then report what you posted.

## Step 1 — Load the review report

Resolve the report file: use the path argument if given, otherwise the newest match of
`"${TMPDIR:-/tmp}"/pr-*-review.md`:

```bash
ls -t "${TMPDIR:-/tmp}"/pr-*-review.md 2>/dev/null | head -1
```

Read it and extract, for every finding in the severity tables (P0–P3): its stable `#`, **file**,
**line**, **severity**, **reviewer(s)**, **confidence**, the one-line issue, and the keyed detail line
(`- **#N** — …`) carrying the why-it-matters and the suggested fix. Also note any **Triage Groups** and
the **Pre-existing** section. If no report file is found, stop and ask for the path.

## Step 2 — Resolve the PR and its diff

Derive the PR ref from the report (its `<ref>` is in the filename and its header names the PR). Confirm
the PR and capture the head commit + the commentable lines:

```bash
gh pr view <ref> --json number,headRefOid,url,state
gh pr diff <ref> --color=never
```

Build the set of **commentable lines** — the `RIGHT`-side (added/context) lines present in the diff per
file. A GitHub inline comment must anchor to a line in the diff; a finding citing a line outside the diff
cannot be posted inline. If the PR is closed/merged, stop and say so.

## Step 3 — Classify each finding

- **Inline** (the default) — the finding's `file:line` is a commentable line in the diff. Post it as an
  inline review comment.
- **Cross-cutting** — the finding (or a Triage Group) is not tied to one line: it spans many files or is
  about the change's overall shape/approach. These are the *only* candidates for a top-level comment, and
  only when genuinely large/cross-cutting.
- **Pre-existing** — do **not** post. These are informational and belong on someone else's earlier work,
  not this PR's threads.
- **Out-of-diff but line-specific** — a line-specific finding whose line isn't in the diff (secondary/
  context code). Skip posting it and list it in your Step 5 report as "not posted (outside diff)"; do not
  inflate it into a top-level comment.

## Step 4 — Post

**Inline comments — one batched review, no top-level body.** Compose a JSON payload and post a single
review with `event: COMMENT` and an empty body, carrying every inline comment:

```bash
# payload.json: {"commit_id":"<headRefOid>","event":"COMMENT","body":"",
#   "comments":[{"path":"<file>","line":<line>,"side":"RIGHT","body":"<comment>"}, ...]}
gh api --method POST "repos/{owner}/{repo}/pulls/<number>/reviews" --input payload.json
```

`event: COMMENT` posts the comments without approving or requesting changes, and an empty `body` means
**no top-level summary**. Pre-filter the `comments` array to commentable lines only (Step 2) so the
review submits cleanly; if a comment is still rejected, drop it and report it rather than failing the
batch.

Each inline comment body is just the finding — concise, professional, present-tense, no change
narration:

```
**P1 · correctness** — <short issue>

<why it matters: what breaks / who's hit, 1–3 sentences>

Suggested fix: <suggested_fix>      (omit this line when the finding has no concrete fix)
```

**Top-level — only if warranted.** If, and only if, there is a genuine large cross-cutting concern (e.g.
a Triage Group spanning the whole change, "this duplicates an existing subsystem"), post exactly one
top-level comment for it via `gh pr comment <ref> --body '…'`. It states the concern — it is not a
summary of the review. If there is no such concern, post nothing top-level.

## Step 5 — Report back

Return a concise summary to the user (not to the PR):
- How many inline comments were posted, and on which files.
- Whether a top-level cross-cutting comment was posted (and why), or none.
- Any findings not posted (outside the diff, or pre-existing), so nothing is silently dropped.
- The PR URL.

## Quality bar

- Inline by default; a finding with a commentable `file:line` is always posted inline, never folded into
  a top-level comment.
- At most one top-level comment, and only for a real large cross-cutting concern. Never a summary,
  verdict, or "here's what I found" comment.
- Never post pre-existing findings.
- Account for every primary finding: posted inline, posted in the cross-cutting note, or reported as
  not-posted with the reason. None silently dropped.
