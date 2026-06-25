# skills

Riley's personal collection of agent skills and resources, packaged as a Claude Code plugin.

## Layout

```
.claude-plugin/
  plugin.json          # plugin manifest (name, description, version, author)
skills/
  hello-world/
    SKILL.md           # one skill per directory
```

Skills under `skills/` are auto-discovered — there's no need to list them in the manifest.

## Adding a skill

Create `skills/<skill-name>/SKILL.md` with YAML frontmatter:

```markdown
---
name: my-skill
description: One line describing what the skill does and when to use it. Claude uses this to decide when to invoke it.
---

# My Skill

Instructions for Claude go here.
```

Keep the `description` specific about *when* to trigger — that's how the skill gets selected.
Supporting files (scripts, references, templates) can live alongside `SKILL.md` in the same
directory.

## Installing locally

Add this repo as a plugin marketplace and install it:

```
/plugin marketplace add /Users/rmoynihan/misc/skills
/plugin install rmoynihan-skills
```

Then verify it works by asking Claude to run the `hello-world` skill.
