---
name: auto-resolve-pr-comments
description: Autonomously resolve a PR's review comments end to end with no human review of the plan — runs plan-pr-comment-responses to produce the response plan, then immediately runs execute-pr-response-plan to implement, validate, verify, push, drive CI green, and post/resolve replies. Use when the user wants a PR's comments handled fully hands-off.
argument-hint: "PR number / URL / branch"
disable-model-invocation: true
---

# Auto-resolve PR comments

Chain the two PR-comment skills into one hands-off run: produce the response plan, then execute it
end to end — **without stopping for the user to review the plan in between.** This skill adds no
behavior of its own; it only orchestrates.

1. **Plan.** Invoke the **`plan-pr-comment-responses`** skill (Skill tool) on the given PR ref to
   generate the response-plan file. Do **not** pause for the user to review it — proceed straight
   to step 2.
2. **Execute.** Invoke the **`execute-pr-response-plan`** skill (Skill tool), pointed at the plan
   file just produced, and let its full pipeline run.

For all actual behavior, mechanics, and guardrails, see those skills:
- `skills/plan-pr-comment-responses/SKILL.md`
- `skills/execute-pr-response-plan/SKILL.md`

Autonomy here means skipping **plan review** — not ignoring genuine blockers. If the executor hits a
hard stop (CI still red after its retry cap, a TODO that can't be implemented as specified, no plan
file found), surface it to the user rather than forcing through.
