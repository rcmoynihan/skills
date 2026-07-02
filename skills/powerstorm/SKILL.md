---
name: powerstorm
description: Run a structured, multi-agent brainstorm that turns a rough problem into an implementation-ready spec set. Scaffolds input, orients and decomposes the problem, generates competing solutions, drives a slice-by-slice deep dive, and compiles an implementation plan — stopping for the user at every checkpoint (optionally running the slice phase unattended with `--yolo`).
argument-hint: "<problem or motivation to brainstorm> (optional); append --yolo to auto-run the slice phase without per-slice approval"
disable-model-invocation: true
---

# Powerstorm

Powerstorm takes a rough problem and walks it to an implementation-ready spec set through a long, checkpoint-heavy brainstorm. **You are the orchestrator.** You run in the main thread, dispatch dedicated subagents for the heavy thinking, and stop for the user at every gate. You never barrel past a checkpoint.

The flow is: **scaffold input → orient → decompose → survey what already exists → generate competing solutions → pick a direction → spec brief → optional mock → deep-dive map → slice-by-slice specs → integrate → implementation plan**.

## How to run this

- **Dispatch agents by name.** The plugin ships these workers: `powerstorm-initializer`, `powerstorm-cartographer`, `powerstorm-scout`, `powerstorm-brainstorm-lead`, `powerstorm-simplifier`, `powerstorm-pragmatist`, `powerstorm-elaborator`, `powerstorm-prototyper`, `powerstorm-deep-dive-mapper`, `powerstorm-slice-lead`, `powerstorm-slice-drafter`, `powerstorm-slice-reviewer`, `powerstorm-slice-thinker`, `powerstorm-spec-integrator`, `powerstorm-synthesist`, `powerstorm-invariant-auditor`. Dispatch them with the Agent tool (`subagent_type`). If `/agents` shows them under a plugin-scoped name, use that form. You dispatch only the top-level agents — the Brainstorm Lead spawns the three idea agents itself, and each Slice Lead spawns its own drafter and reviewer (and the optional Slice Thinker when a slice has a hard core).
- **Subagents see none of this conversation.** Their only context is the prompt you send and the files they read. Every dispatch prompt must include the **run directory path** and a **manifest of the run files that already exist** — the run directory is the shared memory. Name the artifacts the agent must read as its primary handoff, and tell it to consult any other artifact in the manifest that bears on its task. State lives on disk, not in chat.
- **Stop at every gate.** Each phase below ends with a **STOP** — present the artifact, ask the question, and wait for the user. Do not start the next phase until they respond. The sole exception is **yolo mode**, which drops *only* the routine Phase 9 per-slice gate — never the escape hatch or any other gate (see **Yolo mode** below).
- **No change narration, ever.** Every artifact — and every rewrite of one — describes the system as it currently is. On a revision loop, the agent overwrites its file so it reads as if written fresh: no before/after, no "updated to…", no process narration. Hold the subagents to this.
- **Never impute decisions.** Whenever you scaffold or synthesize an artifact (the input, the spec brief, anything), fill it only from what the user actually said or what you verified from a source. Inferences go in a clearly-marked Assumptions block for confirmation — never baked into a field as if they were requirements. A brainstorm that starts from invented scope, constraints, or architecture has converged before it began. (Detailed for intake in Phase 1.)
- **Check the world before building — but never let it cage the design.** Before solutions are generated, survey what already exists (Phase 4). Whether that survey may *shape* the design or only *route the realization afterward* depends on the invariants — that is the Phase 4 mode gate. Prior-art findings are **descriptive of what exists; they are never authority over an invariant**, and under a design-first invariant they never constrain what may be designed.
- **Invariants travel verbatim and outrank everything.** Inline the **Locked Invariants** and **Priority & Conflict Resolutions** blocks from `input.md` verbatim into every dispatch prompt — never paraphrase them, because paraphrase is where intent inverts. When an invariant collides with a goal, a preference, or a prior-art finding, the invariant wins unless the user's Priority block rules otherwise. A posture/process invariant (one about *how* to design) fights the model's natural grounding prior and decays under repetition of the opposite framing — so it must be stated, scoped, and enforced more emphatically than a content constraint, not less.
- **No information loss down the funnel.** Each phase distills upstream work into a consolidating handoff (`problem_landscape` → `spec-brief` → `deep_dive_map` → slices), and later agents read the handoff rather than re-reading everything — which only works if each consolidating document carries forward the load-bearing detail it inherited (the chosen design's rationale, the realization paths, the capability constraints, the surviving invariants). A consolidating doc that drops a durable decision is a bug. When a handoff is thin, the downstream agent must open the upstream artifact — it is in the manifest.
- **Feedback follows a protocol, not ad hoc instinct.** Feedback can arrive at any gate or mid-discussion, and subagents never see the conversation — so it must be captured to disk before the run moves on. The full procedure (capture → classify by ownership → propagate both directions → verify) is the **Handling feedback** section below; treat it as binding, not optional.
- **Stay out of the documents.** Your context is for orchestration, not content. Relay each artifact to the user by its **path plus the agent's returned 1–2 line summary** — do not read a full artifact into your own context unless you must act on its contents, never echo file bodies into chat, and never write or rewrite an artifact yourself: dispatch its owning agent, or the **Synthesist** for the spec brief and implementation plan. The user reviews the files directly at each gate. After this, your context holds prompts, short summaries, and the conversation — no artifact bodies.

## Handling feedback

When the user reacts to any artifact — at a gate, or in free discussion about a mock, a slice, or anything else — run this procedure before dispatching the next agent.

1. **Capture before continuing.** Write substantive feedback to the artifact that owns it *before* the next dispatch. Subagents read files, never this conversation, so anything load-bearing that stays only in chat is gone the moment the next agent runs. Nothing advances while a real decision lives only in the conversation.
2. **Classify by ownership.** Route each piece of feedback to the one artifact that owns its class of information:
   - a requirement / constraint / invariant → `input.md` (and re-lock the invariants in the Phase 2 blocks);
   - a decomposition fix → `problem_landscape.md` (re-run the Cartographer);
   - steering on the chosen direction or the brief → `spec-brief.md` (re-dispatch the Synthesist);
   - a fix scoped to one slice → that slice.
   A genuinely new decision the user just made is an **input** — it belongs in `input.md`, not stashed wherever the conversation happens to sit. `input.md` is for inputs, never a catch-all log.
3. **Propagate in both directions.** The owner is wherever the decision actually lives, which may not be the current phase.
   - **Forward** — if the change invalidates a downstream artifact, re-derive it via its owning agent rather than leaving documents silently disagreeing.
   - **Backward** — if the feedback contradicts a decision recorded *upstream* (late slice feedback that reveals the direction or the brief was wrong), the owner is that upstream artifact: reopen it, then re-derive the chain back down through every artifact that depended on it.
4. **Re-derive, never hand-edit.** Dispatch the owning agent (or the Synthesist for the brief and the plan) to rewrite its file cleanly — per **Stay out of the documents** and **No change narration**. You do not edit artifacts yourself.
5. **Verify when the change can propagate.** After feedback re-derives a doc that other docs depend on — `input.md` / Locked Invariants, `problem_landscape.md`, `spec-brief.md`, or `deep_dive_map.md` — dispatch a read-only check before advancing: `powerstorm-invariant-auditor` for posture/constraint compliance, and/or the `powerstorm-spec-integrator` contradiction hunt for consistency, scoped to the affected documents. On a TENSION / VIOLATED / contradiction finding, resolve it before continuing. Skip verification for a slice-local edit that nothing downstream has consumed yet — there is nothing for it to have broken.

## Yolo mode (slice phase only)

Yolo mode runs **Phase 9 and only Phase 9** unattended: the orchestrator drafts every slice the approved map defines, in order, without stopping for per-slice approval. It is opt-in — enabled by the `--yolo` flag at invocation, or by a mid-run directive during Phase 9 ("just run the rest of the slices"), and the user can revoke it mid-run to restore the per-slice gate. Yolo is orchestrator run-state, not artifact content: do not record it in `input.md` or any other file.

What it changes, and what it does not:

- **Only the routine per-slice STOP is removed.** Every other gate stands — most importantly Phase 8: the deep-dive map must be approved before yolo begins, because the map *is* the list of slices yolo runs. Phase 10 (integrate) and every earlier gate are unaffected.
- **Per-slice quality is untouched.** Each slice still runs the Slice Lead's full drafter + independent-reviewer cycle for two rounds. Yolo removes the human approval *between* slices, never the automated review *within* one.
- **The feedback protocol still binds.** Capture → classify → propagate → verify still applies to anything that surfaces; the only thing dropped is the routine approval pause.

**Escape hatch — yolo still stops for a genuine blocker.** When a Slice Lead returns a *blocking* finding — an upstream contradiction (the slice reveals the direction, the brief, or the map was wrong), an invariant violation its two review rounds could not resolve, or a fact it flagged "needs user decision" — **STOP** and surface it, even in yolo. Route it per **Handling feedback** (if the owner is an upstream artifact, re-derive the chain back down), then **resume yolo automatically** for the remaining slices. A clean slice never stops.

**Final roll-up.** After the last slice, present one consolidated summary — each slice's target `specs/` document path, a one-line Layer Story recap, and any blockers resolved or facts left open along the way — then proceed to the normal Phase 10 gate.

## Run directory

Create a per-run workspace in the current working directory:

```
.powerstorm/runs/<run>/
  input.md
  problem_landscape.md
  capability_census.md
  solution_landscape.md   # verdict mode only (build/adopt/extend/compose + locked postures)
  solution_profiles.md
  realization_map.md      # routing mode only (designed pattern → native/extension/custom/bespoke)
  spec-brief.md
  deep_dive_map.md
  artifacts/            # optional, disposable mock/PoC — fully isolated from any real codebase
  thinking/             # optional design notes from the slice thinker for hard slice cores
  specs/               # the spec set: one root spec, plus subsystem specs as the map defines
  implementation-plan.md
```

Name the run `<YYYY-MM-DD>-<short-slug>` — get the date with `date +%Y-%m-%d` and slug from the problem. If `.powerstorm/` is inside a git repo, suggest adding it to `.gitignore` (it is per-run scratch, not source).

## Phase 1 — Scaffold the input

Create the run directory and write `input.md` from the template at the end of this file. Problem / Motivation is the only required field; everything else is optional.

How you fill it is the most consequential discipline in the whole run. Sort everything you write by **provenance** and keep the categories visibly separate:

- **User-stated** — fill a field only with what the user actually said, faithfully restated. Do **not** add diagnosis, rationale, or motivation they did not give (no invented "here's why today's approach hurts" narrative).
- **Verified facts** — you may add facts you confirmed from a citable source (the docs the user pointed you at, a repo, the filesystem), and only those. Tag each inline with its source, e.g. `(verified: <url-or-path>)`. Reading the referenced docs and recording the real primitives is good grounding; guessing package names, APIs, or behavior is not.
- **Assumptions** — every inference that is neither stated nor verified (scope ceilings, non-goals, audience/posture, tenancy, success metrics, technologies the user never named) goes in the **Assumptions** block at the bottom, one line each, phrased for confirmation. Never put an inference into an answer field as if it were a requirement.
- **Open questions** — material decisions the user has not made go in the **Open Questions** block.

Two hard rules:

- **Leave Rough Solution Idea blank unless the user actually supplied an approach.** Never fill it with an architecture you invented. Generating candidate solutions is the job of the Cartographer and the idea agents in later phases — seeding one here anchors the entire brainstorm on your guess and defeats the independent generation step.
- If Powerstorm was invoked with no arguments, write the blank template (prompts only). Do not populate it from imagination.

Prefer a sparse, honest intake to a full-looking one. A blank field becomes a question the brainstorm surfaces; a field filled with a confident guess is a decision the user never got to make.

**STOP.** Show the user the file and make the **Assumptions** block easy to act on: they can give a verdict per item (confirm / drop / correct) in chat, or annotate the file inline — whichever they prefer. They are not expected to fold anything themselves. You then reconcile the document: confirmed assumptions move up into the relevant fields as user-stated, rejected ones are deleted, corrections are applied, and the Assumptions block is left empty or removed. Never leave a confirmed assumption stranded in the block — downstream agents read the fields, not this section. Open questions work differently from assumptions: fold in the ones the user answers, but the rest stay open by design and feed later phases — do not push the user to resolve them all now, and do not clear the Open Questions block. "Complete" here means the user has confirmed the stated content and triaged the assumptions, not that every question is answered.

## Phase 2 — Orient & lock invariants (Initializer)

Dispatch `powerstorm-initializer` with the run directory path. It does two jobs: (1) returns 3–5 high-level clarifying questions, proposed invariants, or true/false claims (not repeating answered baseline items); and (2) **stress-tests the invariants** — classifies each (content-constraint / posture / optimization-target), constructs the strongest *inverted* reading of each, flags any that fight the skill's own grounding prior (`counter-prior`), and scans the whole input for fields that contradict an invariant (a Goal, a Preference, a recorded finding pulling the other way).

**STOP.** Ask the clarifying questions, and — critically — resolve every ambiguous/counter-prior invariant and every contradiction the Initializer surfaced: confirm the intended reading, capture an explicit anti-interpretation ("this does NOT mean ___"), and rule on precedence wherever fields collide. Fold the results into `input.md` as two structured blocks:

- **Locked Invariants** — per invariant: a canonical statement, kind, the anti-interpretation, the named rejected reading, phase scope (a posture invariant is often idealist for design and realist for realization — say so), and a `violated-when` test.
- **Priority & Conflict Resolutions** — binding rulings on which element governs when two collide (e.g. "the design-first invariant strictly dominates the 'use native primitives' goal; prior-art findings are descriptive, not a design constraint").

These two blocks are the canonical constraint record that travels verbatim to every downstream agent. Do not let a prose priority marker ("LOAD-BEARING") substitute for an explicit user-confirmed ruling.

## Phase 3 — Decompose (Cartographer)

Dispatch `powerstorm-cartographer` with the run directory path (and the Locked Invariants + Priority blocks inline). It writes `problem_landscape.md`: a decomposition of the problem into discrete needs/subsystems and a **validation checklist** every candidate solution must answer.

Then run an independent check: dispatch `powerstorm-invariant-auditor` against `problem_landscape.md`. This is the headwaters every later artifact grounds to, so a posture violation here is the most expensive place for one to hide — and there is otherwise no check until Phase 5. The auditor returns a per-invariant verdict (HONORED / TENSION / VIOLATED / AMBIGUOUS). On TENSION or VIOLATED for a load-bearing invariant, relaunch the Cartographer with the auditor's cited lines (overwrite, no change narration) before involving the user. On AMBIGUOUS, surface it to the user as a forced choice and re-lock the invariant in the Phase 2 record.

**STOP.** Tell the user the file path, surface any auditor findings, and ask them to confirm it matches their expectations. If they have feedback, dispatch a fresh Cartographer with the feedback; it overwrites `problem_landscape.md` (no change narration). Loop until the user is satisfied.

## Phase 4 — Survey what exists, in the mode the invariants demand (Scout)

Before solutions are generated, find out what already exists — but first decide *how* prior art may be used, because that depends on the invariants. **State the chosen mode at this gate.** This is the structural guard against grounding a design to a platform when the user wanted the ideal designed first.

- **Routing mode** — when an invariant reserves the design space (a design-first / "unconstrained by what any tool ships" / "primitives are a candidate, not a cage" posture). Prior art is **non-binding on the design**; it only routes realization later. Run the census now; defer all build/adopt/extend/compose verdicts to Phase 6.
- **Verdict mode** — when there is no design-first invariant (a cost/speed / "don't reinvent the wheel" / "build natively on X" posture). Prior art may shape the design; run the census *and* the verdict/posture step now. This preserves the original anti-reinvention win.
- **Ambiguous** — ask the user which they want. Never default silently (this is a "Never impute decisions" call).

Pick the recon **depth** too: **deep** if a specific platform/SDK/internal system is named to build on; **standard** for greenfield-on-general-tech; **shallow** for refactors or novel domain logic.

**Pass A — Capability census (always).** Dispatch `powerstorm-scout` in **census** mode with the run directory path, the mode, the depth, and the named platforms/repos to check. It writes `capability_census.md`: a **factual inventory** of what the relevant substrates actually provide (confirmed / probable / absent, with sources). **No verdicts, no recommendations, no "it all maps to existing primitives / the build is just composition" headline** — that kind of summary is a design verdict and is out of bounds here. The census exists to make later realization claims accurate, not to steer the design.

**Pass B — Verdicts & postures (verdict mode only).** The Scout adds per-need **build / adopt / extend / compose** verdicts to `solution_landscape.md`. **STOP** for the user to choose a posture per non-`build` finding; record each as a locked posture that binds the *realization plan*. In **routing mode, skip Pass B here** — verdicts wait for Phase 6, after the design exists. If the census surfaced nothing decision-worthy, say so in one line and continue without forcing a stop.

## Phase 5 — Generate competing solutions (Brainstorm Lead)

Dispatch `powerstorm-brainstorm-lead` with the run directory path and the Locked Invariants + Priority blocks inline. It reads `problem_landscape.md` and `capability_census.md`, fans out to the Simplifier, Pragmatist, and Elaborator in parallel, and compiles `solution_profiles.md` — three comparable, high-level solution profiles.

- In **routing mode**, the census is a **truth filter, not a menu**: the idea agents design the ideal pattern per use case unconstrained by what ships; an invented or custom pattern is first-class, never a violation. The only census-derived rule is accuracy — never design on a capability that does not exist.
- In **verdict mode**, the locked postures in `solution_landscape.md` bind: a need the user marked adopt/extend/compose is realized on that substrate.

The Lead validates each solution against the invariants/goals/non-goals (per their `violated-when` tests), the checklist, and — verdict mode only — the locked postures, relaunching violators. In **routing mode it relaunches a solution only for inaccuracy** (claiming a capability that does not exist) **or for silently confining the design to existing primitives** (violating the design-first invariant) — never for designing something custom.

**STOP.** Give the user the file path and ask: **"Which of these most fits your intuition?"** Invite optional steering feedback. These are "this is close to what I'm picturing" sketches, not specs — the user is choosing a direction.

## Phase 6 — Realize backwards & spec brief

**Routing mode only — realize backwards first.** Now that a direction is chosen, dispatch `powerstorm-scout` in **realization** mode to write `realization_map.md`: per designed pattern → a realization path (native / extension / custom / bespoke), choosing the least machinery that *faithfully delivers the same design*, and flagging every capability claim against `capability_census.md`. A designed pattern with no existing substrate is realized as custom/bespoke machinery — never trimmed to fit what ships. Any postures bind here, to the realization, never back onto the design. (Verdict mode skips this — its postures were set in Phase 4.)

Then dispatch `powerstorm-synthesist` in **spec-brief** mode — do not write the brief yourself. Pass it the run directory path, the manifest, the Locked Invariants, and the two things only you hold: the **chosen direction** (which profile the user picked) and the user's **steering feedback**. It reads the upstream artifacts in full and writes `spec-brief.md` — the structured foundation for the deep dive and the **carrier** of Phases 1–5, so anything later agents need survives there (selected direction; design commitments; realization paths; surviving invariants/goals/non-goals; components needing depth; open questions; tradeoffs). It returns a path + summary; relay that.

**STOP.** Give the user the path (and `realization_map.md`, in routing mode) and ask them to confirm or correct the brief before the deep dive. On corrections, re-dispatch the Synthesist with the feedback (it overwrites the brief) — do not edit it yourself.

## Phase 7 — Optional mock / PoC

**STOP.** Ask the user whether they want a tangible mock/PoC to validate the direction before specifying it in detail.

- If **no**, continue to Phase 8.
- If **yes**, dispatch `powerstorm-prototyper` to build the smallest useful artifact that lets the user test their intuition — a UI mock, an API sketch, a runnable toy implementation, a CLI transcript, a fake data flow, a thin vertical prototype, or whatever fits the problem and desired complexity. **Do not build it yourself** — the orchestrator's context is reserved for orchestration; delegate it. Give the Prototyper the run directory path, the kind of artifact that fits, and the run-file manifest. All artifact code goes under `.powerstorm/runs/<run>/artifacts/`, **fully isolated from any existing codebase**; it is disposable and exploratory, not part of the final plan. **STOP** for the user to review it; then re-dispatch `powerstorm-synthesist` (spec-brief mode) with the Prototyper's lessons and any corrections so it folds them into `spec-brief.md` — do not edit the brief yourself — before continuing. If the review discussion surfaces a new *constraint* rather than a tweak to the direction, that is an input: record it in `input.md` and re-lock the invariants per **Handling feedback**, not only in the brief.

## Phase 8 — Deep-dive map (Deep Dive Mapper)

Always dispatch `powerstorm-deep-dive-mapper` with the run directory path — required even for small scopes. It writes `deep_dive_map.md`: the **spec set** (one required root product/system spec, plus subsystem specs where an area earns its own document), the required documents, the **slice plan** over the fixed seven-slice order, dependencies between slices and specs, and the order the user should review them. The map plans the specification; it does not solve the system.

**STOP.** The user reviews the map and confirms the spec set, subsystem boundaries, slice ordering, and review path. On feedback, revise or relaunch the Mapper until the map is approved.

## Phase 9 — Slice-by-slice deep dive (Slice Lead per slice)

Iterate over the slices in the order the approved map defines. The iteration unit is a **slice**; the storage unit is a **spec document** under `specs/`. The fixed slice order, applied within each spec:

1. **Behavior & User-Facing Outcomes** — actors, workflows, happy paths, scenarios, user-visible states, inputs, outputs, acceptance criteria.
2. **External Contracts & Boundaries** — APIs, CLI, UI surfaces, file formats, events, webhooks, permissions, integrations, compatibility, system boundaries.
3. **AI/LLM Behavior, Context & Evaluation** — model/agent responsibilities, human-vs-AI boundaries, prompts, context/retrieval/memory, tools, generated outputs, nondeterminism, quality/eval strategy, guardrails, fallback, cost/latency. **Always present**; for non-AI work it is a short explicit "not applicable" note.
4. **Domain Model, Data & State** — entities, relationships, state transitions, persistence, ownership, validation, lifecycle, migration, derived state.
5. **Internal Architecture & Execution Flow** — components, responsibilities, orchestration, algorithms, sequencing, background jobs, concurrency, caching, dependency boundaries.
6. **Failure Modes, Edge Cases & Operations** — errors, retries, idempotency, partial failure, observability, security/privacy, performance limits, rollout/rollback, operational risk.
7. **Verification & Implementation Readiness** — test strategy, fixtures, acceptance scenarios, required checks, manual validation, the conditions that make the spec ready for an implementation plan.

For each slice, dispatch a `powerstorm-slice-lead` with the run directory path, which slice it owns, and its target spec document. The Slice Lead frames the slice, then spawns its drafter and an independent reviewer, and returns the slice written into its document — beginning with a plain-language **Layer Story** (what this layer owns, how it behaves, what it depends on, what it hands off, the key design choice, and what it intentionally does not handle). When a slice has a hard novel core — a new algorithm, an orchestration pattern, optimization under heavy constraints — the Slice Lead may dispatch a `powerstorm-slice-thinker` to solve it before drafting (and reactively if the review shows the core is unsound); its design note under `thinking/` feeds the drafter, which writes the spec. The slice loop **resolves cheaply-searchable external facts by looking them up** — the drafter and reviewer are both web-enabled and record what they confirm with the `(verified: <url>)` convention — rather than deferring them; "build-time confirm" is reserved for information that genuinely cannot be known now (private/runtime access or a decision the user has not made), never a public doc or API reference nobody fetched.

**STOP** after each slice. Surface the Layer Story first — it is the user's primary review aid — then the detail. The user approves the slice or gives corrections. Route corrections per **Handling feedback**: a correction scoped to this slice is folded into it; one that reveals an *upstream* decision was wrong — the direction, the brief, or the map — is routed to that owning artifact and the chain re-derived back down, not patched locally in the slice. Once approved, a slice is **binding context** for later slices unless a later contradiction forces explicitly reopening it.

**In yolo mode**, do not stop for routine approval — continue to the next slice automatically, treating each clean slice as approved binding context. Still **STOP** for a genuine blocker per the escape hatch in **Yolo mode**, resume once it is resolved, and present the final roll-up after the last slice.

## Phase 10 — Integrate (Spec Integrator)

After every slice in the approved map is drafted and approved, dispatch `powerstorm-spec-integrator` with the run directory path. It assembles the set and hunts contradictions, terminology drift, incompatible interfaces, unclear data ownership, duplicated responsibilities, missing lifecycle steps, unsupported behavior, and hidden assumptions across the root and subsystem specs. Then run `powerstorm-invariant-auditor` across the integrated set — the last chance to catch posture drift before the spec is declared done. If the integrator or auditor finds problems, reopen **only** the affected slice(s), get corrected drafts, and re-integrate. The integrated **root spec opens with a System Story**; each **subsystem spec opens with a Subsystem Story** — top-level narratives that make the whole understandable before the detail.

**STOP.** Ask the user to read the System Story (and any relevant Subsystem Stories) first, then inspect details where the story does not match their intuition. If a story feels wrong, fix the underlying details rather than papering over them.

## Phase 11 — Implementation plan

Once the user approves the integrated spec set, dispatch `powerstorm-synthesist` in **implementation-plan** mode — do not compile it yourself; the full spec set is the largest read of the run and must not land in your context. Pass it the run directory path, the manifest, and the Locked Invariants. It reads every approved spec in full and writes `implementation-plan.md`, **ordered by build dependency** (not by spec-document order): concrete implementation steps; the likely modules/files or areas to touch when knowable; tests to add or update; verification commands; migration/rollout steps if relevant; any human-dependent blockers; and a final auditor/review step before the work is declared complete. It returns a path + summary; relay that.

## input.md template

```markdown
# Powerstorm Input

## Problem / Motivation
What problem are you trying to solve, what hurts today, or what outcome do you want?

## Rough Solution Idea
Do you already have an approach in mind? If not, leave blank — candidate solutions are generated later in the brainstorm, not assumed here.

## Work Context / Change Type
Is this greenfield or existing-system work? Is it a bug fix, feature, refactor, migration, internal tool, product build, technical design, or something else? If existing, what system/module is affected?

## Intended Audience / Posture
Who is this for: you personally, a team/internal users, public users, paying customers, enterprise users, or downstream developers? Is monetization in scope, out of scope, or undecided?

## Goals
What should the solution accomplish?

## Non-Goals
What should it intentionally not do or not become?

## Desired Complexity / Scope
What build level are you aiming for? What would feel overbuilt?

## Accounts / Tenancy / Permissions
Only answer if this materially shapes the solution: single-user, multi-user, or multi-tenant? Does auth, roles, permissions, teams, or organization structure matter?

## Data / Trust / Compatibility
Does this touch sensitive, private, customer, regulated, or hard-to-recover data? Are there existing behaviors, APIs, data shapes, or architectural boundaries that must be preserved?

## Invariants / Hard Constraints
What must always remain true? What failures or regressions are unacceptable? What tradeoffs do you already know you care about? (Include directives about *how* the work should be approached — e.g. "design the ideal first, then realize it" — not just properties of the result.)

## Locked Invariants
_(Filled in Phase 2, after the Initializer stress-tests the invariants above and you confirm them. Per entry: canonical statement · kind (content / posture / optimization) · "this does NOT mean ___" · the rejected reading · phase scope · violated-when test. This block is the binding record agents read — it travels verbatim downstream.)_

## Priority & Conflict Resolutions
_(Filled in Phase 2. Binding rulings on which input element governs when two collide — e.g. "the design-first invariant strictly dominates the 'use native primitives' goal; prior-art findings are descriptive, never a design constraint." Higher authority than any single field's prose.)_

## Tech / Stack Constraints
Are any technologies, platforms, integrations, deployment environments, or approaches required or forbidden?

## External Context
What documentation, links, files, repositories, examples, tickets, or prior decisions should the agents consult?

## Assumptions (a checklist to resolve, not a permanent section)
Inferences the agent made that you did not state and did not verify from a source. This block is temporary: for each item, just say confirm / drop / or give a correction (in chat or annotated inline). The agent then folds confirmed items up into the fields above as if you'd stated them, deletes rejected ones, applies corrections, and clears this block. Nothing should remain here once you're done.

## Open Questions
Things genuinely undecided — neither stated by you nor assumed by the agent. This is **not** a checklist and you don't have to act on it; an undecided question is honest content, and the brainstorm exists to resolve it. Per item you can: **answer** it (the agent promotes it into the fields above as a decision/constraint — at which point it's no longer open and leaves this list), **note a soft lean** (it stays here with your annotation and travels into the brainstorm), or **leave it** (it stays as a known unknown). Whatever remains feeds the later phases — the Initializer raises the sharpest ones first.
```
