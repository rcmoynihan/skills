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

Skills that produce artifacts — review reports, PR response plans, grill / swarm-code
run state, idea briefs, specs, prior-art assessments, and design docs — write them to a
single dedicated dir under the OS temp dir:

```
${TMPDIR:-/tmp}/code-goblin-pro/
```

`${TMPDIR:-/tmp}` is the base (honoring a per-user `$TMPDIR` on macOS, falling back to `/tmp`); the
`code-goblin-pro/` subdir namespaces every skill's files in one place, keyed by run (e.g.
`grill-<slug>/`, `swarm-code-<date>-<slug>/`, `pr-<ref>-review.md`). Any new skill that writes
scratch or deliverable files uses this dir. One deliberate exception: `survey-terrain` writes its
terrain pack to `<repo>/.grill/terrain.md` (gitignored) — the pack must outlive tmp cleanup and be
discoverable per repo.

## Skills

| Skill | Invoke with | What it does |
|---|---|---|

### Discovery, specification, and delivery

| Skill | Invoke with | What it does |
|---|---|---|
| `survey-terrain` | `/survey-terrain` | Maps an existing product, codebase, or organization into a durable terrain pack at `<repo>/.grill/terrain.md`. |
| `opp-grill` | `/opp-grill` | Interviews an unformed problem or opportunity, tests its evidence, evaluates directions, and ends at a user-owned go/no-go/defer/adopt gate. |
| `to-idea` | `/to-idea` | Compiles an opportunity grill into a standalone idea brief. |
| `spec-grill` | `/spec-grill` | Interviews the product decisions—the what—while parking technical choices for the design lane. |
| `to-spec` | `/to-spec` | Compiles a product grill into a testable PRD/spec with invariants, requirements, and acceptance criteria. |
| `prior-art-check` | `/prior-art-check` | Evaluates existing internal, open-source, and commercial options against a compiled spec before design work. |
| `design-grill` | `/design-grill` | Interviews the technical decisions—the how—using the compiled spec as locked input. |
| `to-design` | `/to-design` | Compiles a design grill and its spec into a technical design document. |
| `swarm-code` | `/swarm-code` | Delivers a settled spec, design, or plan to a review-ready branch through a verified multi-agent implementation flow. |

### Review and pull-request workflow

| Skill | Invoke with | What it does |
|---|---|---|
| `code-review` | `/code-review` | Produces a multi-persona, report-only review of a branch or pull request. |
| `post-review-comments` | `/post-review-comments` | Posts a completed review report's findings to a pull request, primarily as inline comments. |
| `review-and-comment` | `/review-and-comment` | Runs a review and posts its findings without pausing for report review. |
| `plan-pr-comment-responses` | `/plan-pr-comment-responses` | Turns pull-request review feedback into a comment-by-comment response and implementation plan. |
| `execute-pr-response-plan` | `/execute-pr-response-plan` | Implements an approved response plan, verifies it, drives CI green, and posts and resolves applicable replies. |
| `learn-from-pr-comments` | `/learn-from-pr-comments` | Proposes focused code-review-profile improvements from review feedback. |
| `auto-resolve-pr-comments` | `/auto-resolve-pr-comments` | Runs the PR-comment planning, execution, and learning flow end to end, pausing only before profile edits. |

### Agent operation and communication

| Skill | Invoke with | What it does |
|---|---|---|
| `drive-claude-code` | `/drive-claude-code` | Delegates coding tasks to the Claude Code CLI from the shell. |
| `drive-codex` | `/drive-codex` | Delegates coding tasks to the OpenAI Codex CLI with `codex exec`. |
| `agent-orchestration` | `/agent-orchestration` | Helps choose and coordinate an agent architecture, from a direct model call through multi-agent orchestration. |
| `coding-agent-pitfalls` | `/coding-agent-pitfalls` | Identifies common AI coding-agent failures and the guardrails that avoid them. |
| `prompt-engineering` | `/prompt-engineering` | Guides prompt design for direct calls, tool-using agents, and multi-agent systems. |
| `handoff` | `/handoff` | Compacts the current conversation into a handoff document for another agent. |
| `interrupt-resume` | `/interrupt-resume` | Resumes a task interrupted before completion. |
| `state-it-plainly` | `/state-it-plainly` | Restates a previous response in concise, jargon-free language. |
| `hello-world` | `hello world` | Verifies that the plugin is installed and working. |

Skills with `disable-model-invocation: true` are slash-command only. The other skills may also be
selected by Claude when their descriptions match the request; every skill remains available as its
listed slash command.

## Opportunity-to-design flow

Use this flow when the work begins with a problem or opportunity rather than a settled product idea:

```
/opp-grill → /to-idea → /spec-grill → /to-spec → /design-grill → /to-design
```

`/opp-grill` establishes whether a problem is worth pursuing and ends with a user-owned verdict. On
a `go` verdict, `/to-idea` produces the idea brief that `/spec-grill` uses to settle the product
intent. `/to-spec` compiles that work into the product contract. `/design-grill` then resolves the
technical realization against that locked contract, and `/to-design` compiles the resulting
technical design document.

Start at `/spec-grill` when the product idea is already clear. For an existing codebase or
organization, `/survey-terrain` can provide orientation before either grill. After `/to-spec`, run
`/prior-art-check` when a build-versus-adopt decision needs evidence; a `build` or
`adopt-with-gaps` verdict proceeds to `/design-grill`. `/swarm-code` can implement the settled
design after `/to-design`.

Several skills dispatch plugin agents by name. `code-review` uses the `code-review-*` review panel;
the grill skills share `grill-*` workers; `swarm-code` uses `swarm-code-*` leads and workers; and
`to-design` uses the `to-design-*` hardening panel.

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
