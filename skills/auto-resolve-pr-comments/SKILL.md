---
name: auto-resolve-pr-comments
description: Autonomously resolve a PR's review comments end to end with no human review of the plan — runs plan-pr-comment-responses to produce the response plan, then immediately runs execute-pr-response-plan to implement, validate, verify, push, drive CI green, and post/resolve replies, then runs learn-from-pr-comments to feed the lessons back into our code-review profiles (pausing once for approval of those edits). Use when the user wants a PR's comments handled fully hands-off.
argument-hint: "PR number / URL / branch"
disable-model-invocation: true
---

# Auto-resolve PR comments

Chain the PR-comment skills into one run: produce the response plan, execute it end to end
**without stopping for the user to review the plan in between**, then feed the lessons back into our
own review. This skill adds no behavior of its own; it only orchestrates.

1. **Plan.** Invoke the **`plan-pr-comment-responses`** skill (Skill tool) on the given PR ref to
   generate the response-plan file. Do **not** pause for the user to review it — proceed straight
   to step 2.
2. **Execute.** Invoke the **`execute-pr-response-plan`** skill (Skill tool), pointed at the plan
   file just produced, and let its full pipeline run.
3. **Learn.** Once execution finishes, invoke the **`learn-from-pr-comments`** skill (Skill tool),
   pointed at the same plan file, to turn the reviewers' catches into de-duplicated rule additions
   for our `code-review` profiles. This step **pauses for the user to approve** the proposed profile
   edits before applying any — the one deliberate human gate in the run.

For all actual behavior, mechanics, and guardrails, see those skills:
- `skills/plan-pr-comment-responses/SKILL.md`
- `skills/execute-pr-response-plan/SKILL.md`
- `skills/learn-from-pr-comments/SKILL.md`

Hands-off applies to the **resolution** — steps 1–2 run without pausing for plan review. It does not
mean the run never stops: step 3 always pauses for approval before changing our own review rules,
and the executor surfaces genuine blockers rather than forcing through (CI still red after its retry
cap, a TODO that can't be implemented as specified, no plan file found).
