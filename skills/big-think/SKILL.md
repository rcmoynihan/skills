---
name: big-think
description: Diagnose the root cause of a hard, not-yet-understood problem, then diverge and converge on a recommended approach. A diagnosis-first thinking harness — triage the problem (Cynefin), frame it, fan out parallel diagnostic lenses to build competing cause hypotheses, force the leading one through a hard Root-Cause Gate (falsifiable, red-teamed) before any solutioning, then brainstorm remedies mapped to the cause and converge on one recommendation with tradeoffs. Standalone — ends at a decision record, not a spec. Use for a stubborn bug, regression, outage, or a non-obvious "why is this happening?" problem (often software, sometimes not) where the cause is genuinely unknown and jumping to solutions is the risk.
argument-hint: "<the hard problem / symptom to diagnose> (optional)"
disable-model-invocation: true
---

# big-think

big-think takes a hard problem whose *cause you don't yet understand* and works it through two diamonds: first to a **locked, evidence-backed root cause**, then to a **recommended approach**. **You are the orchestrator.** You run in the main thread, dispatch subagents for the heavy diagnostic and adversarial thinking, and — the discipline that defines this skill — **you do no solutioning until the root cause has survived the Root-Cause Gate.**

The flow: **triage & frame → diverge on causes → Root-Cause Gate → diverge on remedies → converge & record.**

It is diagnosis-first, software-first but general, and **standalone**: it ends at a decision record — a recommended approach with its tradeoffs — not a spec, and it does not hand off to another skill.

## How to run this

- **Dispatch agents by name.** The plugin ships five workers: `big-think-scout` (grounds one question across all surfaces + runs the gate's read-only probe), `big-think-diagnostician` (one diagnostic lens, dispatched once per lens), `big-think-red-team` (attacks the leading root-cause hypothesis at the gate), `big-think-idea` (generates one remedy, dispatched once per stance), and `big-think-premortem` (attacks the chosen remedy). Dispatch them with the Agent tool (`subagent_type`); if `/agents` shows them plugin-scoped, use that form. Workers never spawn workers — you own all fan-out.
- **You are the sole author of the run's files.** Workers are read-only advisors that return structured briefs; you synthesize them into `problem.md`, `hypotheses.md`, `root-cause.md`, `remedies.md`, and `decision-record.md`. This keeps one coherent hypothesis tree and stops parallel lenses colliding on a shared file.
- **Subagents see none of this conversation.** Their only context is the prompt you send and the files they read. Every dispatch prompt must carry the **run directory path**, the framed problem (or the locked cause) **inline**, and the worker's exact output contract. State lives on disk and in the prompt, not in chat.
- **Two hard gates, otherwise autonomous.** Within a phase you run without stopping. You STOP for the user at exactly two points: the **Root-Cause Gate** (confirm the cause before any remedy work) and the **final recommendation** (the decision and its tradeoffs). A third, lighter stop is conditional — only when the problem's framing or scope is genuinely ambiguous at intake. Don't add gates the user didn't ask for; don't skip the two that are load-bearing.
- **Diagnosis before solutioning is not negotiable.** Phase 3 (remedies) is *procedurally invalid* until Phase 2 emits a root cause that meets the gate's exit criteria and has survived the red team. If nothing survives, the run's honest output is an **investigation plan**, not a remedy. Never let your own hunch stand in for a gated hypothesis.
- **Establish scope, persona, and constraints from what the user actually said.** Sort intake by provenance — user-stated / verified-from-a-source / assumption / open-question — and never bake an inference into the problem statement as if it were given. A diagnosis that starts from an invented scope has converged before it began. (Detailed in Phase 0.)
- **Right-size the fan-out to the problem.** The Cynefin class plus "is this a concrete failing system?" decide how many diagnostic lenses fire and which. A Clear problem may need none (answer it). A Complicated bug fires two or three lenses. A gnarly multi-system problem fires more. Fanning out every lens on every problem is the anti-pattern — three focused lenses beat five scattered ones.
- **Generation and selection stay in separate hands.** You never both brainstorm and pick. Idea agents generate; a scored convergence with adversarial gates selects. Collapsing the two is how a diamond stops being a diamond.
- **One researcher for everything off the working tree.** Reading the local tree is every agent's job. Anything beyond it — org repos (`gh`), deployed/live systems (`kubectl` / `psql` / `aws` / `snow` / logs / metrics), or external prior art — goes through `big-think-scout`, which carries the read-only, staging-by-default discipline. A single grounding task may weave across surfaces in one pass. When a web question is a genuine multi-source survey, invoke the `deep-research` skill instead of the scout.
- **No change narration.** Every artifact describes the problem and the analysis as they currently stand. On a re-run, overwrite the file so it reads as written fresh — no before/after, no "updated to…".
- **Stay out of the documents.** Relay each artifact to the user by its path plus a 1–2 line summary; don't echo file bodies into chat. Your context is for orchestration and synthesis, not for holding whole documents.
- **Loop back when the evidence says the frame was wrong.** If the red team dismantles every hypothesis, or the pre-mortem kills every remedy, that is a signal the framing or the cause was wrong — reopen the gate or the frame, don't patch forward. (The "if three fixes fail, stop — it's mis-framed" instinct, made structural.)
- **Cap the loops.** Diagnosis rounds, gate red-team passes, and convergence are bounded: if a phase can't converge in about two or three rounds, surface the impasse to the user — as an investigation plan or a decision under acknowledged uncertainty — rather than looping forever.

## Run directory

Create a per-run workspace in the current working directory:

```
.bigthink/runs/<YYYY-MM-DD>-<slug>/
  problem.md          # framed problem, scope/persona, Cynefin class, evidence ledger
  hypotheses.md       # competing cause hypotheses — scored, cross-linked, with evidence
  root-cause.md       # the locked cause + gate dossier   (OR investigation-plan.md if the gate fails)
  remedies.md         # candidate remedies, each mapped to the causal link it addresses
  decision-record.md  # the deliverable: recommended approach + tradeoffs (ADR-shaped)
```

Name the run `<YYYY-MM-DD>-<slug>` — get the date with `date +%Y-%m-%d`, slug from the problem. If `.bigthink/` is inside a git repo, suggest adding it to `.gitignore` (it is per-run scratch, not source).

## The diagnostic lens playbook

Phase 1 fans out `big-think-diagnostician` workers, each dispatched with **one lens** — the analytical method it applies. You choose which lenses fire and how many from the Cynefin class and whether the problem is a concrete failing system. Each lens generates candidate causes *independently* (workers don't see each other) and returns structured hypotheses. Fire the lenses that fit, not all of them.

- **historian** *(almost always)* — "what changed?" Reconstruct the timeline — deploys, merges, config/flag flips, dependency bumps, data/traffic shifts, infra events — against when the symptom began. Software's highest-yield first question.
- **boundary** *(concrete failing system)* — the seams: inputs, API/version contracts, assumptions that hold on one side of an interface and not the other, error propagation across components.
- **kepner-tregoe** *(concrete failing system with a sharp symptom)* — the IS / IS-NOT specification (What / Where / When / Extent): what exhibits the problem versus what could but doesn't. The true cause must explain every IS *and* every IS-NOT; candidates that contradict an IS-NOT are eliminated.
- **fault-tree** *(failure with several contributing conditions)* — take the symptom as the top event and decompose through AND/OR gates into the basic conditions that could produce it; find the minimal combinations.
- **fishbone / systems** *(messy, multi-factor, or non-software)* — sweep candidate causes across categories (code / config / data / dependencies / infra / process, or the domain's real categories) so no whole class is missed; for systems problems, map the stocks / flows / feedback loops and name the dominant loop.
- **first-principles** *(when the framing itself is suspect)* — strip the problem to fundamentals and ask whether an assumed constraint is real; use when the obvious causes are excluded and the problem "shouldn't be happening."

A **Complex** problem (Cynefin) replaces analysis-to-a-cause with **safe-to-fail probes**: instead of lenses that reason to a cause, dispatch the Scout to run small read-only experiments and observe, then converge on what the probes reveal.

## Phase 0 — Triage & frame

Create the run directory and write `problem.md` from the template below. Fill it by provenance: what the user stated, what you verified from a source (tagged), what you are assuming (flagged, never baked into a field), what is still open. Establish scope and persona explicitly — do not silently import them.

Then do three things:

1. **Classify with Cynefin** and record it — it dials the whole run.
   - **Clear** (known, ordered): a solved kind of problem. Say so and give the known answer / runbook; don't spin up the machine.
   - **Complicated** (knowable by analysis): the default hard-engineering path — deterministic root-cause analysis via the lens fan-out.
   - **Complex** (cause visible only in retrospect): you can't analyze your way to it. Replace lens analysis with parallel safe-to-fail probes and converge on what they reveal.
   - **Chaotic** (crisis, no stable cause yet): recommend the stabilizing action first, then re-triage the calmed problem.
2. **Check the frame** (cheap, one pass): is this the right problem? Whose problem is it, what is the gap between desired and perceived, is there a better problem to solve? A quick inversion — "what would guarantee this keeps happening?" — often exposes the real target. Reframe before diagnosing if the frame is off.
3. **Ground the symptom if needed** — dispatch `big-think-scout` once when the reported symptom should be confirmed against reality (a live dashboard, a log, "is this already known in the org?") before you trust the framing.

**STOP — only if the framing or scope is genuinely ambiguous.** Present `problem.md`, resolve the ambiguity, and fold the answers in. If the problem and scope are already clear, do not stop — proceed to Phase 1.

## Phase 1 — Diverge on causes

Draft the candidate-cause structure in `hypotheses.md`: a MECE-ish issue tree of where the cause could live, so the fan-out covers the space without overlap or gaps.

Fan out `big-think-diagnostician` workers — one per chosen lens (see the playbook; count gated by Cynefin and system-concreteness, typically two to four). Each dispatch prompt carries the run directory path, the framed problem inline, the **lens and its method**, the **slice of the issue tree** it owns, and the output contract (structured hypotheses: candidate cause · mechanism · evidence-for · evidence-against · confidence · the discriminating test that would confirm or refute it). Workers read the local tree directly and **name off-tree questions in their return** rather than reaching into org repos or live systems themselves — you route those to the Scout so five lenses don't hit prod in parallel.

Merge the returned hypotheses into `hypotheses.md` as first-class, scored, cross-linked entries: dedup convergent causes, and note where lenses independently agree — agreement raises confidence. This is the "Discover" diverge; you converge it at the gate.

## Phase 2 — Root-Cause Gate  *(hard gate)*

Rank the hypotheses by evidence and discriminating power (differential diagnosis: prefer the next check that best *separates* the leaders; rule out anything an IS-NOT or a probe contradicts; keep a "can't-miss" set of high-severity causes explicitly excluded, not silently dropped). Run the cheap discriminating tests — dispatch the Scout to execute the read-only probe (query the DB, pull the metric/log window, inspect deployed config) that confirms or refutes the leader. **The gate is a probe against reality, not just an argument.**

Then attack the leader: dispatch `big-think-red-team` (one or two independent passes) to argue it is *just a symptom*, find what it fails to explain, audit whether the evidence truly supports the mechanism, and name the decisive outstanding test.

**Exit criteria — the leading hypothesis passes the gate only if ALL hold:**

- **Mechanism** — a stated causal chain from cause to symptom, not a correlation.
- **Evidence for and against** — both recorded; the against side doesn't sink it.
- **Explains affected *and* unaffected** — accounts for what shows the symptom and what doesn't (the IS / IS-NOT test).
- **Falsifiable** — a concrete test whose outcome would disprove it, and whose actual result is consistent with the cause.
- **Actionable · Preventable · Fundamental · Verifiable** — the four-property root-cause test: something you can act on that would prevent recurrence, not a symptom.
- **Confidence** — a stated level, and it survived the red team.

- **If it passes** → write `root-cause.md` (the locked cause + the dossier above).
- **If nothing passes** → write `investigation-plan.md`: the surviving hypotheses and the specific discriminating tests/probes to run next to tell them apart. This is a legitimate, honest terminal outcome — **not** a fake solution.

**STOP (required).** Present `root-cause.md` (or `investigation-plan.md`) and get the user's confirm / correct. Nothing proceeds to remedies without a confirmed cause. If the run ended in an investigation plan, this is where it ends.

## Phase 3 — Diverge on remedies

*(Reachable only through a confirmed gate.)* Reframe the locked cause as one or more **How-Might-We** questions. Fan out `big-think-idea` workers — one per **stance** (typically minimal-intervention · structural/root-fix · lateral/reframe) — each dispatched with the locked root cause inline, the run directory path, its stance, and the constraint that **every proposed remedy names the causal link it breaks** and honors the problem's scope and constraints. Generation only — the idea agents don't rank or pick.

Compile the returned remedies into `remedies.md` as comparable profiles: approach · which causal link it breaks · cost/complexity · risks & reversibility · what it deliberately doesn't fix.

## Phase 4 — Converge & record

Score the surviving remedies against weighted criteria — does it break the causal link · cost/effort · risk · reversibility · blast radius — a simple decision matrix; where it's close, weigh the criteria deliberately rather than by first impression. Before locking the leader, two cheap adversarial gates:

- dispatch `big-think-premortem`: "this remedy shipped and the problem came back, or a new one appeared — why?" Fold the top failure modes into the risks.
- run a **second-order** check yourself: "and then what?" — does a later-order effect reverse the benefit?

Converge on **one** recommended approach (or a tight ranked pair if it is genuinely too close to call), and write `decision-record.md` (template below): the problem, the locked root cause and its confidence, the recommended approach and why, the alternatives considered and why they lost, the key risks (including the pre-mortem findings and any second-order effects), and the open questions. For a software problem it is ADR-shaped; for a non-software problem it is a plain decision record. **It recommends an approach; it does not prescribe running another tool.**

**Loop-back:** if the pre-mortem kills every remedy, or scoring shows nothing actually breaks the causal link, the cause or the frame was wrong — reopen the gate (Phase 2) or the frame (Phase 0). Don't ship a remedy that doesn't address the cause.

**STOP (required).** Present `decision-record.md` by path plus summary, surfacing the recommendation and its tradeoffs for the user's decision.

## Templates

### problem.md

```markdown
# big-think · <problem slug>

## Symptom / Problem
What is observed to be wrong or stuck — the observable, concrete and specific, not a guess at the cause.

## Cynefin class
Clear | Complicated | Complex | Chaotic — and one line of why. This dials the run.

## Scope & persona
Whose problem this is, what system/area is in scope, what is explicitly out. (User-stated; inferences flagged.)

## Desired vs. perceived
What "fixed" looks like, and the gap between the desired and the current state.

## Evidence ledger
What we know, each tagged: (stated) / (verified: <path-or-url>) / (assumed — confirm) / (open).
- ...

## Constraints
Hard limits on any remedy — compatibility, don't-touch-X, budget, deadline. User-stated.

## Assumptions to confirm
Inferences made that were not stated or verified — one line each, for the user to confirm / drop / correct.

## Open questions
Genuinely undecided; these feed the diagnosis.
```

### hypotheses.md

```markdown
# Candidate causes · <problem slug>

## Issue tree
Where the cause could live (MECE-ish), so the search covers the space.
- ...

## Hypotheses

### H1 — <one-line candidate cause>
- **Lens(es):** which lens(es) surfaced it; note cross-lens agreement
- **Mechanism:** cause → … → symptom
- **Evidence for:** … (tagged with source)
- **Evidence against / doesn't explain:** …
- **Discriminating test:** the check that would most confirm or refute it
- **Confidence:** low | medium | high — on what basis

### H2 — …
```

### root-cause.md

```markdown
# Root cause · <problem slug>

## Locked cause
<one-paragraph statement of the root cause>

## Dossier (gate criteria)
- **Mechanism:** the causal chain, cause → symptom.
- **Evidence for:** … (sourced)
- **Evidence against, and why it doesn't sink it:** …
- **Explains affected & unaffected:** what shows the symptom and what doesn't.
- **Falsifiable test & its result:** the test that could disprove it; what running it showed.
- **Root-cause test:** Actionable / Preventable / Fundamental / Verifiable — each satisfied how.
- **Confidence:** level, and how the red team was answered.

## Ruled out
The rejected hypotheses and the evidence or probe that eliminated each.
```

### investigation-plan.md  *(when the gate does not pass)*

```markdown
# Investigation plan · <problem slug>

_The gate did not pass: no hypothesis met the exit criteria. This is the plan to get there — not a remedy._

## Surviving hypotheses
The candidates still in contention, with current confidence.

## Discriminating tests to run
Each: the test/probe · which hypotheses it separates · what result implicates which · cost/access needed.
1. ...

## What blocks a confident diagnosis now
Missing access, data, or a decision — and what would unblock it.
```

### decision-record.md

```markdown
# Decision record · <problem slug>
_Status: recommended — for your decision_

## Context
The problem, and the **locked root cause** (confidence: …). See `root-cause.md`.

## Recommended approach
What to do, and **which causal link it breaks**. Why this one over the alternatives.

## Alternatives considered
Each remedy weighed, and why it lost — didn't address the cause / cost / risk / reversibility.

## Consequences & risks
Positive and negative outcomes; the pre-mortem's top failure modes and any second-order effects; what this deliberately does not fix.

## Open questions
What remains unresolved or needs the user's call before acting.
```
