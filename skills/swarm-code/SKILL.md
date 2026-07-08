---
name: swarm-code
description: Drive an already-settled spec, design, or completed plan-mode plan (from Claude Code or Codex) to a review-ready branch through a three-phase multi-agent swarm — plan, deliver, review-and-refine — keeping the orchestrator's context pristine and every task independently verified before it integrates. The fourth link after grillmaster → to-spec → to-design, and the on-ramp for a native plan-mode plan; consumes its input as a spec and derives its own task plan, never re-opening design. Use when you have a detailed spec or an approved plan and want it implemented with swarm-grade context isolation and generator/verifier separation.
argument-hint: "(optional) path to a spec/design file, a completed plan-mode plan (e.g. ~/.claude/plans/…), or freetext task; append --draft-pr to also open a draft PR on the deliverable branch"
disable-model-invocation: true
---

# swarm-code

Implementing a large, detailed spec or plan with one coding agent degrades as the work grows: a single context window accumulates every file read, every failed test, and every correction, and the agent self-reviews its own work — inheriting its own blind spots. swarm-code takes an **already-settled** spec/design (or a completed plan-mode plan from Claude Code or Codex) and drives it to merge-ready code through a three-phase, multi-agent swarm, so that **no single context ever holds the whole job** and **every task is independently verified** before it can integrate. It is the fourth link in the suite chain — **grillmaster → to-spec → to-design → swarm-code** — and it also takes a native plan-mode plan directly; either way it consumes its input as a settled spec and never re-opens product or design decisions.

**You are the orchestrator, and your context is the asset to protect.** You run in the main thread and dispatch exactly three lead subagents, one per phase, in order: `swarm-code-plan-lead` (Phase 1), `swarm-code-delivery-lead` (Phase 2), and `swarm-code-review-and-refine-lead` (Phase 3). You hold **only file paths and one-line summaries** — never artifact bodies, diffs, plan contents, or worker transcripts. The heavy context (the plan, the diffs, the reasoning, the failed attempts) lives inside the leads and their children, and on disk in a run directory that acts as shared memory. This is the primary value of the skill, not an implementation detail: context hygiene is what a single agent cannot give you, and it is enforced structurally by the topology and the run dir, not by convention. If you find yourself reading a diff or a plan body into your own context, you are defeating the skill — read the path, dispatch the lead that owns that work, and keep the pointer.

The primary value, in priority order: (1) **context hygiene / single-responsibility agents** — the load-bearing property above; (2) **quality via generator/verifier separation** — every task gets an independent correctness check and an independent pitfall audit, in fresh context, never self-review; (3) **throughput** — a conditional bonus only where the plan proves disjoint file ownership, never the justification for running.

## Accepted inputs and on-ramps

swarm-code's input is a **settled statement of what to build**, in any of these forms — all consumed identically as the run's **input spec** (a path the leads read, never a body you hold):

- a spec/design artifact from the suite chain (**grillmaster → to-spec → to-design → swarm-code**);
- a **completed plan-mode plan** from Claude Code's or Codex's built-in plan mode — the approved output of a native planning session;
- a freetext task stated directly at invocation.

The plan-mode on-ramp is a first-class peer of the suite chain, not a special case — it is the natural bridge for someone who planned in Claude Code or Codex and now wants that plan executed with swarm-grade context isolation and generator/verifier separation.

**Reference the plan file in place — never rewrite or re-summarize it.** Claude Code persists each plan-mode plan as a standalone markdown file under `~/.claude/plans/`, named `<slug>-<adjective>-<noun>.md` (that trailing `<adjective>-<noun>` is how a user refers to one — e.g. "the giggly-kite plan" is `…-giggly-kite.md`). Resolve the path and hand it down as the input-spec path; you rewrite nothing and no plan body enters your context or a lead's prompt.

```bash
ls -t ~/.claude/plans/*.md 2>/dev/null | head              # most-recent plan-mode plans
ls -t ~/.claude/plans/*giggly-kite*.md 2>/dev/null | head  # by the <adjective>-<noun> name the user gives
```

Referencing the existing file is the canonical path. Only when the plan is **not** already a file on disk — a Codex plan that lives in session state, or a plan pasted inline — does the main thread capture it verbatim to `input-plan.md` in the run dir and use that path instead. Either way, `spec_source` records the path used.

**A plan-mode plan is an input spec, never swarm-code's own `implementation-plan.md`** — two different artifacts that share a word. The incoming plan states *implementation intent* (steps, files, sequencing) but carries none of swarm-code's coordination structure: no task DAG, no waves, no four-tier ownership matrix, no per-task verification modes, no verbatim invariants block. So swarm-code **does not start writing code on receipt of a plan** and **does not skip Phase 1.** It runs Phase 1 in full — the scout grounds the plan, the intake gate applies the same under-specification test to it (being approved in plan mode is *not* the gate's sufficiency verdict; a native plan can be approved while a product or design decision is still unmade, which STOPs here exactly as any spec would), and the plan-lead derives the DAG, waves, ownership, and verification gates into `implementation-plan.md` — the artifact the user approves at the one interactive gate.

## One mechanism, scaled by width

There is exactly **one** flow, parameterized by how many tasks run in parallel. A fully sequential run — **swarm-of-one**, where every wave is width 1 — is the *degenerate case of the same mechanism*, not a separate code path. A small task and a large one are handled identically; the small one simply scales down to swarm-of-one with no worktrees and no sub-branches.

There is **no task-size gate, no over-use nudge, and no decline for being small or coupled.** Invoking swarm-code runs it as intended at any size. Quality (independent verification) is orthogonal to parallelism, so even a one-task run benefits from the swarm's generator/verifier separation. Never warn the user that their task is "too small" to swarm, and never suggest they run something else instead. The only condition that hard-declines a run is under-specification (below); size, coupling, parallelizability, and cost never do.

## What swarm-code does not do

- It never **auto-merges.** The deliverable is a `swarm/<slug>` branch (and, on request, a draft PR) that the user reviews and merges themselves. The user is always the final gate.
- It never touches the user's **original branch history.** All run commits land on `swarm/<slug>`; the original branch receives none.
- It never **edits the input spec.** It diagnoses why a spec is under-specified (the ambiguity register) and routes the user onward, but it does not rewrite product or design decisions — those belong to grillmaster / to-spec / to-design.
- It has **no cost, token, budget, or wall-clock accounting** of any kind. Runaway protection is entirely progress-based (attempt caps, repeated-command and no-progress aborts, and a re-plan cap). Do not track or report tokens, cost, or budget anywhere.

## Dispatching the three leads

Dispatch each lead with the Agent tool, using its bare `subagent_type` (`swarm-code-plan-lead`, `swarm-code-delivery-lead`, `swarm-code-review-and-refine-lead`); if `/agents` shows them plugin-scoped, use that form instead. You dispatch these three and **only** these three. Each lead spawns its own children — the plan-lead its scout/gate/reviewer, the delivery-lead its task-leads and integrator, the review-and-refine-lead the reused `code-review-lead` plus fix task-leads and integrator — but that nesting is theirs to manage. "Exactly three" governs what *you* dispatch; it does not cap the total agent count. Nesting is bounded at depth three, and leaf agents cannot delegate.

Every dispatch you issue is **self-contained**: the lead sees no conversation history and receives no artifact bodies inline. You hand it the run-dir path, the file manifest it needs (which run-dir files to read and, for a fresh run, the input-spec path), and its one job with an explicit return contract. What comes back is a set of file paths plus a one-line summary — never a diff, a plan body, or a transcript. When a lead's work carries the spec's invariants forward, those invariants travel **verbatim** — copied from the input spec into the plan's `invariants:` field and from there into every worker prompt, never paraphrased — because paraphrase is exactly where intent inverts.

## Run directory & resuming

All run state lives under a dedicated dir in the system temp dir; **nothing is written to the repo tree except the `swarm/<slug>` branch.** Use `${TMPDIR:-/tmp}/code-goblin-pro` in shell (the suite convention).

```
${TMPDIR:-/tmp}/code-goblin-pro/swarm-code-<date>-<slug>/
  run-state.json          # slug, integration_branch, original_branch, stash_ref, current_phase, current_wave
  input-plan.md           # captured input spec — present only when a plan-mode plan was pasted inline instead of referenced as a file
  scout-inventory.md      # the scout's factual codebase/prior-art brief (grounds the intake gate)
  ambiguity-register.md   # STOP output (empty/absent on PROCEED)
  implementation-plan.md  # THE contract — the approved copy is immutable to workers
  status/<task-id>.json   # per-task resume checkpoint
  worktrees/<task-id>/     # concurrent-task worktrees (width>1 only); absent for swarm-of-one
  code-review/            # Phase-3 staging: full.diff, files.txt, intent.md, review-round-1.md
  logs/                   # per-agent dispatch/return logs (the coordination trace)
```

`<slug>` derives from the task/spec. The run dir is the observability surface: the user inspects a run by reading these files, and each phase and lead reads from and writes to it as shared memory. It is retained on success (for inspection) and on abort; it is never auto-deleted.

**Before starting a new run, check for an existing one** — a swarm run is long and will outlive a compaction:

```bash
ls -dt "${TMPDIR:-/tmp}"/code-goblin-pro/swarm-code-*/ 2>/dev/null
```

If the invocation identifies an existing run — a run-dir path or `<slug>`, or an explicit resume request — **resume** it from the on-disk state; otherwise start a fresh run. This is not an interactive checkpoint: the plan-approval gate is the one place swarm-code pauses for the user, so resume is a routing decision you make from the invocation and the run-dir check, not a prompt you put to the user. Resume rebuilds position entirely from on-disk state — `run-state.json` (the current phase and wave), the per-task `status/<task-id>.json` checkpoints, and the `swarm/<slug>` branch itself (including the recorded auto-stash ref) — and continues from the **first incomplete wave**. Committed work is safe regardless of the run dir, because it lives on the durable branch; only the plan/status metadata is volatile, so resume is best-effort within a session. Do not re-plan or re-run completed waves; read their status and carry on.

## The git hub: auto-stash, branch, restore

swarm-code works in the user's **main working tree** on a dedicated branch, and guarantees the user's original branch is untouched. The lifecycle has four concrete steps, and you (the main thread) own the invocation, completion, and abort steps; the leads do the during-run committing on the branch.

1. **On invocation** — if the working tree is dirty, **auto-stash** the uncommitted *and* untracked changes and record the stash ref in `run-state.json`. Then check out `swarm/<slug>` **off HEAD**. The plan is built against HEAD post-stash, so any stashed work-in-progress is *not* part of the run's basis.
2. **During the run** — commits land on `swarm/<slug>` in the main tree (sequential delivery), on `swarm/<slug>/<task-id>` sub-branches (concurrent delivery, each in its own worktree), or on `swarm/<slug>/<fix-id>` sub-branches (Phase-3 fixes — sequential, built in the main tree, no worktree); all are serial-merged into `swarm/<slug>`. The original branch never receives a commit.
3. **On completion** — restore the main tree to the user's original branch and pop the stash. The deliverable is the `swarm/<slug>` branch. When `--draft-pr` was passed, after a successful run you (the main thread) open the optional additional ending point: push `swarm/<slug>` and run `gh pr create --draft` with the base set to the user's original branch (the branch they will merge into). This opens a **draft** PR only — it never auto-merges, and the user remains the final merge gate. **Never auto-merge.**
4. **On abort** — remove any extra worktrees, restore the original branch, pop the stash, and **preserve** `swarm/<slug>` and the run dir so the user can inspect or resume.

Restoration is conflict-free on the normal path because the original branch received no run commits; the recorded stash ref is what makes restoration recoverable even after an interruption.

## The three-phase flow

The main thread orchestrates three phases in order, with exactly one interactive gate between Phase 1 and Phase 2. Phases 2 and 3 run autonomously once the plan is approved.

### Phase 1 — Planning

Dispatch `swarm-code-plan-lead` with the run-dir path and the input-spec path — a spec/design artifact, a plan-mode plan file (referenced in place; see *Accepted inputs and on-ramps*), or the freetext task, all consumed identically. The plan-lead runs Phase 1 end to end inside its own context: a **scout** produces a factual codebase/prior-art inventory; an **intake-gate** applies the governing under-specification test grounded in that inventory; on PROCEED the plan-lead derives the task DAG, the parallel waves (wave 0 = contract/setup), the four-tier per-task ownership matrix, per-task and per-wave verification gates, and any `requires_human` flags; and a **plan-reviewer** runs the three credibility tests before you ever see the plan. This full derivation runs even when the input is itself a plan-mode plan — that plan is the spec Phase 1 derives *from*, never the `implementation-plan.md` it produces. What the plan-lead hands back to you is the path to `implementation-plan.md` (or the path to `ambiguity-register.md` on a STOP) plus a one-line summary — not the plan body.

### The plan-approval gate — the one interactive contract

This is the **single** point where swarm-code pauses for the user. Present the path to `implementation-plan.md` and a short summary, and **stop.** Nothing downstream runs until the user responds.

- **Approve** → Phase 2 (Delivery) and Phase 3 (Review-and-Refine) proceed **autonomously to completion**. Do not gate again except at a `requires_human` task or an unrecoverable failure (below).
- **Corrections** → hand the corrections back to `swarm-code-plan-lead`, which re-derives the plan, and **re-present** the revised plan at this same gate.
- **Decline** → the run ends having **executed zero code.** Declining the plan is the intended dry-run: it exercises intake, planning, and review with no risk, because no code is written before approval.

A material semantic-conflict kickback during delivery (when the integrator cannot compose two parallel branches without a behavioral choice) re-derives the affected tasks via the plan-lead and **re-enters this same gate** with the revised plan — it is not a new kind of halt. Only a *material* change (scope/ownership) re-presents; a minor re-derivation continues without re-approval.

### Phase 2 — Delivery (autonomous)

After approval, dispatch `swarm-code-delivery-lead` with the run-dir path and the approved plan path. The delivery-lead executes the plan wave by wave, wave 0 first, entirely within its own subtree:

- A **width-1 wave** runs one task-lead in the main tree on `swarm/<slug>`, committing straight there — no worktree, no sub-branch.
- A **width>1 wave** runs each task concurrently in its own worktree and `swarm/<slug>/<task-id>` sub-branch, with task-leads in parallel.
- Each task runs a maker-checker loop (worker implements; independent correctness-verifier and pitfall-auditor evaluate in fresh context), and after a wave's tasks verify, the integrator serial-merges them in dependency order, running the wave gate after each merge.

You do not see any of this fan-out. The delivery-lead returns the integration state (the wave reached, the branch tip) as paths and one-liners.

### Phase 3 — Review-and-Refine (autonomous)

Dispatch `swarm-code-review-and-refine-lead` with the run-dir path and the plan path. It stages the integrated diff and spawns the reused `code-review-lead` against it (in local scope, against the main tree on `swarm/<slug>`), turns findings into scoped fix task-leads (the same maker-checker loop), and has the integrator re-merge the fixes. This is a **single review pass** — it does not re-review after the fixes merge; it halts only if a fix cannot pass or the integrator kicks back a semantic conflict, never silently shipping an open P0/P1. It returns the review report path plus a verdict one-liner.

## The under-specification STOP

This is **distinct from** the approval gate: the approval gate is a *pause* (the run is healthy and waiting for a go-ahead); the under-specification STOP is a *decline* (the run cannot responsibly start). Under-specification is the **only** hard STOP — the sole condition that declines a run outright.

The intake-gate applies one governing test, grounded in the scout's factual inventory: *"if I started implementing right now, would I be guessing about the spec, invariants, or requirements?"* If yes, it STOPs. Both missing decisions (**gaps**) and conflicting ones (**contradictions**) are subtypes of under-specification. A gap that is genuinely *inferable from the codebase or the spec's own logic* is normal implementation latitude and yields PROCEED — the gate distinguishes real under-specification from ordinary latitude by grounding the judgment in what the codebase already shows.

On STOP, surface to the user: the path to `ambiguity-register.md`, a short summary of what blocks, and the **named routing skill** for each blocker — `to-spec` when the missing piece is *what* to build, `to-design` when it is *how*, or `grillmaster` when the idea needs a rethink. **No code is written.** swarm-code diagnoses the incompleteness; it does not resolve it and never edits the spec.

## Halt surfaces

Beyond the approval gate, only two conditions halt an otherwise-autonomous run:

- A **`requires_human` task** — one the plan flagged as verifiable only by human sign-off (runtime/visual/external-system judgment no agent can confirm). This is a high bar, reserved for exactly that; it pauses for the user's sign-off.
- An **unrecoverable failure** — a task that cannot pass after its attempts and one effort-tier escalation, an ownership veto, or a review-and-refine fix that cannot pass or that the integrator rejects as a semantic conflict.

On either, **halt rather than degrade.** Finish all *independent* work first (sibling tasks already in flight in the same wave are disjoint, so they complete and hand up), then halt at the **dependency frontier** and report the blocked task/wave plus the current run state. Never proceed past a broken dependency with a gap, and **never silently ship open P0/P1 findings.**

## Inspect, abort, resume

The run dir is the observability surface, and the main thread surfaces only paths and one-line summaries to the user (never bodies). Three user actions:

- **Inspect** — the user reads the run-dir files (`run-state.json` for phase/wave, `status/<task-id>.json` for per-task progress, the plan, the register, `logs/`) to see the run's state at any time.
- **Resume** — re-invoking swarm-code on the same run picks up from the first incomplete wave using on-disk state (above).
- **Abort** — triggers the abort lifecycle above: remove extra worktrees, restore the original branch, pop the stash, and preserve `swarm/<slug>` plus the run dir.

## Guardrails for this skill

- **Suite boundary.** swarm-code references only `grillmaster`, `to-spec`, and `to-design` as sibling skills — these are the STOP-routing targets. Reusing `code-review` (via `code-review-lead`) and drawing the pitfall rubric's provenance from `coding-agent-pitfalls` are dependency relationships, not sibling references. Never route to, mention, or delegate to any other suite skill.
- **No cost/token machinery.** There is no token, cost, budget, metering, or ceiling anywhere. All bounds are progress-based; do not add resource accounting even as a convenience.
- **All artifacts are current-state.** The plan, the reports, and the worker-generated code describe the system as it is — no before/after, no "we changed," no process narration. Task branches read as final state, squashed and tidy, not as the journey that produced them.
- **Agents are model- and effort-pinned per role.** Each agent's frontmatter sets the `model` (Opus or Sonnet 5) and the `effort` tier its role's reasoning demand warrants, rather than inheriting the session model. Opus runs the roles whose judgment is load-bearing and costly to get wrong — the plan-lead and plan-reviewer, the intake gate, the correctness-verifier, the pitfall-auditor, and the delivery-lead. Sonnet 5 runs the roles that are mechanical or tightly bounded by their contract — the scout, the worker, the integrator, the task-lead, and the review-and-refine-lead. `effort` tiers the reasoning budget within each.
