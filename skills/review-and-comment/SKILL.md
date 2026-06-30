---
name: review-and-comment
description: Autonomously review a branch/PR and post the findings as PR comments end to end, with no human review of the report in between — runs code-review to produce the review report, then immediately runs post-review-comments to post the findings inline (top-level only for a genuine cross-cutting concern, never a summary). Use when the user wants a PR reviewed and commented fully hands-off.
argument-hint: "blank for current branch/PR, or a PR number / URL / branch"
disable-model-invocation: true
---

# Review and comment

Chain the two review skills into one hands-off run: produce the review report, then post it to the PR —
**without stopping for the user to read the report in between.** This skill adds no behavior of its own;
it only orchestrates.

1. **Review.** Invoke the **`code-review`** skill (Skill tool) on the given branch/PR ref to generate the
   review report file. Do **not** pause for the user to read it — proceed straight to step 2.
2. **Comment.** Invoke the **`post-review-comments`** skill (Skill tool), pointed at the report file just
   produced, and let it post the findings.

For all actual behavior, mechanics, and guardrails, see those skills:
- `skills/code-review/SKILL.md` — how the review is scoped, run, and written.
- `skills/post-review-comments/SKILL.md` — how findings are posted: **strongly inline**, at most one
  top-level comment for a genuine large cross-cutting concern, and **never** a summary / "here's the
  review" comment.

Autonomy here means skipping the **report review** — not ignoring genuine blockers. If a step hits a hard
stop (no PR to comment on, no usable diff, an empty report with no findings, the PR is closed/merged, or a
posting failure), surface it to the user rather than forcing through.
