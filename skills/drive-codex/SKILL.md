---
name: drive-codex
description: Delegate coding tasks to the OpenAI Codex CLI (codex exec) from the shell for building features, refactoring, PR reviews, and batch issue fixing. Use when the user asks to drive, orchestrate, or delegate work to Codex as a sub-agent.
disable-model-invocation: true
---

# Codex CLI

Delegate coding tasks to [Codex](https://github.com/openai/codex) by running its CLI through the Bash tool. Codex is OpenAI's autonomous coding agent CLI.

## When to use

- Building features

- Refactoring

- PR reviews

- Batch issue fixing

Requires the codex CLI and a git repository.

## One-Shot Tasks

Run `codex exec` with the Bash tool. It runs the task and exits when done. Append `< /dev/null` so a non-interactive run never blocks waiting on open stdin:

```
codex exec 'Add dark mode toggle to settings' < /dev/null
```
Set the Bash tool's `timeout` generously for longer tasks, and pass the working directory by prefixing `cd`:

```
cd ~/project && codex exec 'Add dark mode toggle to settings' < /dev/null
```
For scratch work (Codex needs a git repo):

```
cd "$(mktemp -d)" && git init && codex exec 'Build a snake game in Python' < /dev/null
```

## Background Mode (Long Tasks)

For long-running work, launch the command with the Bash tool's `run_in_background: true`, then monitor and control it with the background-shell tools:

```
cd ~/project && codex exec --full-auto 'Refactor the auth module'
```
- Launch it with `run_in_background: true` — the Bash tool returns a shell ID.

- Read its progress with the **BashOutput** tool (returns new output since the last read).

- Stop it with the **KillShell** tool when needed.

Because the Bash tool allocates no PTY, a background `codex exec` cannot be sent interactive keystrokes mid-run. Use `--full-auto` (or `--yolo`) so Codex never stops to ask. If you genuinely need to answer Codex's prompts interactively, run it inside tmux instead and drive it with `tmux send-keys` / `tmux capture-pane` (see the Claude Code skill's interactive pattern).

## Key Flags

| Flag | Effect |
| --- | --- |
| exec "prompt" | One-shot execution, exits when done |
| --full-auto | Sandboxed but auto-approves file changes in workspace |
| --yolo | No sandbox, no approvals (fastest, most dangerous) |
| --sandbox danger-full-access | No Codex sandbox; useful when the host environment breaks bubblewrap |

## Sandbox Caveat

In restricted or containerized environments, Codex `workspace-write` sandboxing may fail even when the same command works in your normal interactive shell. A typical symptom is bubblewrap/user-namespace errors such as `setting up uid map: Permission denied` or `loopback: Failed RTM_NEWADDR: Operation not permitted`.

In that case, prefer:

```
codex exec --sandbox danger-full-access "<task>" < /dev/null
```
Use process boundaries as the safety layer instead: an explicit working directory, a clean git status before launch, narrow task prompts, `git diff` review, targeted tests, and confirmation before committing broad changes.

## PR Reviews

Clone to a temp directory for safe review:

```
REVIEW=$(mktemp -d) && git clone https://github.com/user/repo.git $REVIEW && cd $REVIEW && gh pr checkout 42 && codex review --base origin/main < /dev/null
```

## Parallel Issue Fixing with Worktrees

```
# Create worktrees (run from ~/project)
cd ~/project && git worktree add -b fix/issue-78 /tmp/issue-78 main
cd ~/project && git worktree add -b fix/issue-99 /tmp/issue-99 main
```
Launch Codex in each worktree as a separate background shell (Bash tool with `run_in_background: true`):

```
cd /tmp/issue-78 && codex --yolo exec 'Fix issue #78: <description>. Commit when done.'
cd /tmp/issue-99 && codex --yolo exec 'Fix issue #99: <description>. Commit when done.'
```
Monitor each background shell with the **BashOutput** tool. After completion, push and open PRs:

```
cd /tmp/issue-78 && git push -u origin fix/issue-78
gh pr create --repo user/repo --head fix/issue-78 --title 'fix: ...' --body '...'
```
Clean up the worktrees when done:

```
cd ~/project && git worktree remove /tmp/issue-78
```

## Batch PR Reviews

```
# Fetch all PR refs (run from ~/project)
cd ~/project && git fetch origin '+refs/pull/*/head:refs/remotes/origin/pr/*'
```
Review multiple PRs in parallel — launch each as a background shell (`run_in_background: true`):

```
cd ~/project && codex exec 'Review PR #86. git diff origin/main...origin/pr/86'
cd ~/project && codex exec 'Review PR #87. git diff origin/main...origin/pr/87'
```
Read each review with **BashOutput**, then post results:

```
cd ~/project && gh pr comment 86 --body '<review>'
```

## Rules

- **Git repo required** — Codex won't run outside a git directory. Use `mktemp -d && git init` for scratch

- **Use `exec` for one-shots** — `codex exec "prompt"` runs and exits cleanly; append `< /dev/null` so it never blocks on stdin

- **`--full-auto` for building** — auto-approves changes within the sandbox

- **Background for long tasks** — launch with the Bash tool's `run_in_background: true` and monitor with **BashOutput**; stop with **KillShell**

- **No PTY in background runs** — the Bash tool has no PTY, so use `--full-auto`/`--yolo` to avoid interactive prompts; if you must answer prompts, drive Codex inside tmux

- **Don't interfere** — poll output and be patient with long-running tasks

- **Parallel is fine** — run multiple Codex background shells at once for batch work
