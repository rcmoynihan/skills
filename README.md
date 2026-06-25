# skills

Riley's personal collection of agent skills and resources, packaged as a Claude Code plugin.

## Layout

```
.claude-plugin/
  plugin.json          # plugin manifest (name, description, version, author)
skills/
  <skill-name>/
    SKILL.md           # one skill per directory
```

Skills under `skills/` are auto-discovered — there's no need to list them in the manifest.

## Skills

| Skill | Invoke with | What it does |
|---|---|---|
| `drive-claude-code` | `/drive-claude-code` | Delegate coding tasks to the Claude Code CLI from the shell. |
| `drive-codex` | `/drive-codex` | Delegate coding tasks to the OpenAI Codex CLI (`codex exec`). |
| `handoff` | `/handoff` | Compact the current conversation into a handoff doc for another agent. |
| `vulnerability-scan` | `/vulnerability-scan` | Scan the org's GitHub repos for exposure to a given CVE/advisory. |
| `hello-world` | `hello world` | Minimal smoke test that confirms the plugin is installed. |

All skills except `hello-world` set `disable-model-invocation: true`, so they only fire when you
invoke them explicitly as a slash command — Claude won't trigger them on its own.

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
/plugin install rmoynihan-skills
```

Then verify it works by asking Claude to run the `hello-world` skill.
