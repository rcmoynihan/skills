---
name: code-review
description: Multi-persona code review of a branch or PR. Resolves the changes (current branch's PR, or a given PR / branch), spawns a review-lead subagent that runs always-on plus diff-warranted reviewer personas in parallel, and writes a single markdown review report to the system temp dir. Report-only — never edits, commits, pushes, or posts.
argument-hint: "blank for current branch/PR, or a PR number / URL / branch"
disable-model-invocation: true
---

# Code review

Produce a thorough, multi-perspective code review of a set of changes and hand back a markdown report.
You orchestrate only: resolve the scope, stage the diff, spawn **one `code-review-lead` subagent**, and
surface the file path it returns. The lead runs the whole reviewer fan-out as its own children, so this
thread stays clean.

**Report-only.** Nothing here edits code, commits, pushes, posts to the PR, or resolves threads. The
deliverable is the markdown file.

## Step 1 — Resolve scope and the diff

The argument is empty (review the current branch/PR), or a PR number / URL / branch name. Resolve the
repo and pull what you need with `gh` and `git`. Keep prompts to a minimum by combining commands.

Resolve PR metadata when a PR is involved:

```bash
gh repo view --json nameWithOwner
gh pr view <ref-or-blank> --json number,title,body,url,state,baseRefName,headRefName,headRefOid,isCrossRepository,files,reviews,comments
```

- **Closed/merged PR** → stop with that reason; don't review.
- **No PR found** (bare branch, or no PR for the current branch) → review the branch diff against its
  base; that is still valid.

Pick the **scope mode** — this controls whether reviewers may read the working tree:

- **`local`** — the working tree *is* the reviewed code: no argument, or the argument names the current
  branch, or HEAD already matches the PR head (`headRefName` == current branch, not cross-repository, and
  `git merge-base --is-ancestor <headRefOid> HEAD` succeeds). Compute
  `BASE=$(git merge-base HEAD <baseRefName-or-default>)`, then diff the working tree:
  `git diff -U10 $BASE` and `git diff --name-only $BASE`.
- **`remote`** — reviewing a PR or branch that is **not** the current checkout. Do **not** check it out.
  Use `gh pr diff <ref> --color=never` (or, for a branch, `git diff -U10 $(git merge-base <base> <ref>) <ref>`
  after a `git fetch`). Best-effort fetch the head so reviewers can inspect it:
  `git fetch --no-tags origin <headRefName>:refs/review/<ref>-head` and pass that as `head_ref`.

Resolve the base branch from `baseRefName`, else the repo default branch (never assume `main`). Keep the
merge-base as `review_base`.

## Step 2 — Stage inputs to a run dir

So child prompts pass paths, not large inline blobs, write the resolved scope into a run dir:

```bash
RUN_DIR="${TMPDIR:-/tmp}/code-review-<ref>"   # <ref> = PR number or sanitized branch name
mkdir -p "$RUN_DIR"
```

Write `full.diff` (the diff), `files.txt` (changed-file list), and `intent.md` (a 2-3 line intent
summary derived from the PR title/body, commits, and conversation) into `$RUN_DIR`.

## Step 3 — Spawn the review lead

Resolve the absolute path of the findings contract — `references/findings-contract.md` next to this
SKILL.md (i.e. `${CLAUDE_PLUGIN_ROOT}/skills/code-review/references/findings-contract.md`).

Spawn a single **`code-review-lead`** subagent (Agent tool) and pass it:

- `run_dir` and the paths of `full.diff`, `files.txt`, `intent.md`
- `contract_path` (the absolute findings-contract path)
- `scope_mode` (`local` / `remote`), `head_ref` (when set), `review_base`
- the PR number/title/body and whether it has prior review comments (empty for a bare branch)
- `output_path` = `${TMPDIR:-/tmp}/pr-<ref>-review.md`

The lead selects the panel, dispatches the reviewers in parallel, merges/dedups/gates their findings,
self-verifies the P0/P1 findings, writes the report to `output_path`, and returns one line (path +
verdict + counts).

## Step 4 — Surface the result

Print to the user **only**: the returned report file path and the lead's 2-3 line summary (verdict,
finding counts by severity, which reviewers ran). The file is the deliverable; the chat gets the pointer.
This is one-shot — do not stop for interactive checkpoints, and do not act on the findings (no edits, no
commit, no PR post); that is the user's call, or a job for the PR-response skills.

## If inputs are incomplete

If you cannot obtain a usable diff (no PR, no resolvable branch, empty diff), stop and say exactly what is
missing rather than spawning the lead on nothing.
