# skills

Riley's personal collection of agent skills and resources, packaged as a Claude Code plugin.

## Layout

```
.claude-plugin/
  plugin.json          # plugin manifest (name, description, version, author)
agents/
  <agent>.md           # custom subagents a skill dispatches by name (flat — no subdirs)
skills/
  <skill-name>/
    SKILL.md           # one skill per directory
```

Skills under `skills/` and agents under `agents/` are auto-discovered — there's no need to list
them in the manifest.

## Where skills write artifacts

Skills that produce artifacts — review reports, PR response plans, grillmaster / swarm-code /
big-think run state, specs, and design docs — write them to a single dedicated dir under the OS
temp dir:

```
${TMPDIR:-/tmp}/code-goblin-pro/
```

`${TMPDIR:-/tmp}` is the base (honoring a per-user `$TMPDIR` on macOS, falling back to `/tmp`); the
`code-goblin-pro/` subdir namespaces every skill's files in one place, keyed by run (e.g.
`grillmaster-<slug>/`, `swarm-code-<date>-<slug>/`, `pr-<ref>-review.md`). Any new skill that writes
scratch or deliverable files uses this dir.

## Skills

| Skill | Invoke with | What it does |
|---|---|---|
| `drive-claude-code` | `/drive-claude-code` | Delegate coding tasks to the Claude Code CLI from the shell. |
| `drive-codex` | `/drive-codex` | Delegate coding tasks to the OpenAI Codex CLI (`codex exec`). |
| `handoff` | `/handoff` | Compact the current conversation into a handoff doc for another agent. |
| `big-think` | `/big-think` | Understanding-first thinking harness — triage a hard problem into a posture (diagnose / frame / orient / survey / decide), fan out lenses to build a locked understanding through a hard Understanding Gate, then diverge on approaches and converge on a recommended decision record. |
| `code-review` | `/code-review` | Multi-persona review of a branch/PR — a review-lead spawns always-on plus diff-warranted reviewer subagents in parallel and writes a markdown report to the plugin's temp dir. |
| `post-review-comments` | `/post-review-comments` | Post a finished code-review report's findings to the PR — strongly inline, top-level only for a genuine cross-cutting concern, never a summary. |
| `review-and-comment` | `/review-and-comment` | Run `code-review` then `post-review-comments` back-to-back, hands-off — review a branch/PR and post the findings with no report review in between. |
| `hello-world` | `hello world` | Minimal smoke test that confirms the plugin is installed. |

`hello-world` is model-invocable — Claude pulls it in automatically when relevant. The rest set
`disable-model-invocation: true`, so they only fire when you invoke them explicitly as a slash
command.

`code-review` ships the `code-review-*` agents that the skill dispatches by name: the skill spawns
`code-review-lead`, which spawns the reviewer-persona workers as its own children. `big-think` ships
the `big-think-*` workers (`analyst`, `scout`, `red-team`, `idea`,
`premortem`) that its main-thread orchestrator dispatches by name across the two diamonds.

## Adding a skill

Create `skills/<skill-name>/SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill
description: One line describing what the skill does and when to use it.
disable-model-invocation: true   # optional: slash-command only, never auto-triggered
---

# My Skill

Instructions for Claude go here.
```

If you allow model invocation (omit the flag), keep the `description` specific about *when* to
trigger — that's how Claude decides to select it. Supporting files (scripts, references, templates)
can live alongside `SKILL.md` in the same directory.

## Installing locally

Add this repo as a plugin marketplace and install it:

```
/plugin marketplace add /Users/rmoynihan/misc/skills
/plugin install code-goblin-pro
```

Then verify it works by asking Claude to run the `hello-world` skill.
