---
name: big-think
description: Work a hard problem you don't yet understand well enough to solve through two diamonds — first to a locked, evidence-backed understanding of it, then to a recommended approach — doing no solutioning until the understanding survives a hard, red-teamed, reality-probed Understanding Gate. An understanding-first thinking harness with five postures — **diagnose** the root cause of a bug / regression / outage / "why is this happening?"; **frame** a problem you must invent an approach for or don't know how to start (its precise spec + tractability); **orient** in an unfamiliar domain (a validated map); **survey** how a problem is solved at industry / state-of-the-art (a decision-serving landscape); **decide** among largely-known options (a validated decision frame). Triage picks the posture, fans out parallel lenses to build the understanding, gates it, then brainstorms approaches and converges on one recommendation with tradeoffs. Standalone — ends at a decision record, not a spec. Use for a hard problem where jumping to a solution before you understand it is the risk.
argument-hint: "<the hard problem to work through> (optional)"
disable-model-invocation: true
---

# big-think

big-think takes a hard problem you *don't yet understand well enough to solve* and works it through two diamonds: first to a **locked, evidence-backed understanding** of it, then to a **recommended approach**. What "understanding" means depends on the **posture** — the root *cause* of a failure, the *precise shape and tractability* of a problem you must invent an approach for or don't know how to start, the *lay of the land* in an unfamiliar domain, the *state-of-the-art map* of how it's solved, or the *decision frame* for choosing among options. **You are the orchestrator.** You run in the main thread, dispatch subagents for the heavy generative and adversarial thinking, and — the discipline that defines this skill — **you do no solutioning until the understanding has survived the Understanding Gate.**

The flow: **triage & select posture → frame → diverge on understanding → Understanding Gate → diverge on approaches → converge & record.**

It is understanding-first, software-first but general, and **standalone**: it ends at a decision record — a recommended approach with its tradeoffs — not a spec, and it does not hand off to another skill.

## How to run this

- **Dispatch agents by name.** The plugin ships five workers: `big-think-scout` (grounds one question across all surfaces + runs the gate's read-only probe), `big-think-analyst` (one lens, dispatched once per lens), `big-think-red-team` (attacks the leading understanding at the gate), `big-think-idea` (generates one approach, dispatched once per stance), and `big-think-premortem` (attacks the chosen approach). Dispatch them with the Agent tool (`subagent_type`); if `/agents` shows them plugin-scoped, use that form. Workers never spawn Claude subagents — you own all fan-out. (A worker may consult an external model such as Codex as a read-only cross-check; that is a tool call, not a spawned agent.)
- **You are the sole author of the run's files.** Workers are read-only advisors that return structured briefs; you synthesize them into `problem.md`, `hypotheses.md`, `understanding.md`, `approaches.md`, and `decision-record.md`. This keeps one coherent understanding and stops parallel lenses colliding on a shared file.
- **Validate every worker brief before folding it in.** A return that is empty, off-contract, or non-responsive gets one re-dispatch with the same contract; if it still doesn't deliver, record the gap and proceed only if the remaining evidence still supports the phase — never treat a missing brief as evidence of absence.
- **Subagents see none of this conversation.** Their only context is the prompt you send and the files they read. Every dispatch prompt must carry the **run directory path**, the framed problem (or the locked understanding) **inline**, and the worker's exact output contract. State lives on disk and in the prompt, not in chat.
- **Two hard gates, otherwise autonomous.** Within a phase you run without stopping. You STOP for the user at exactly two points: the **Understanding Gate** (confirm the understanding before any solutioning) and the **final recommendation** (the decision and its tradeoffs). A third, lighter stop is conditional — only when the posture, framing, or scope is genuinely ambiguous at intake. Don't add gates the user didn't ask for; don't skip the two that are load-bearing.
- **Understanding before solutioning is not negotiable.** Phase 3 (approaches) is *procedurally invalid* until Phase 2 emits an understanding that meets the gate's exit criteria and has survived the red team. If nothing survives, the run's honest output is an **open-questions plan**, not an approach. Never let your own hunch stand in for a gated understanding.
- **Every gate is a probe against reality — at every posture.** No understanding passes on argument alone: the Scout runs a read-only probe against its load-bearing assumption, and the understanding must **constrain** (rule out some tempting solution paths), **predict** (state something checkable, whose check held), and **direct** (name what any viable approach must address). A locked understanding that can't do all three isn't gated. This is the discipline that keeps big-think from decaying into a generic brainstorm as the postures get fuzzier than a falsifiable root cause.
- **Establish scope, persona, and constraints from what the user actually said.** Sort intake by provenance — user-stated / verified-from-a-source / assumption / open-question — and never bake an inference into the problem statement as if it were given. An analysis that starts from an invented scope has converged before it began. (Detailed in Phase 0.)
- **Right-size the fan-out to the problem.** The posture plus the problem's specifics decide how many lenses fire and which. A Clear problem may need none (answer it). A well-bounded problem fires two or three lenses. A gnarly multi-part problem fires more. Fanning out every lens on every problem is the anti-pattern — three focused lenses beat five scattered ones.
- **Generation and selection stay in separate hands.** You never both brainstorm and pick. Idea agents generate; a scored convergence with adversarial gates selects. Collapsing the two is how a diamond stops being a diamond.
- **The scout owns org-wide and live-system research, and fires at every gate.** Reading the local tree is every agent's job, and any worker may make a *bounded* public-doc or web lookup to verify a mechanism. But anything that reaches **org repos** (`gh`) **or deployed/live systems** (`kubectl` / `psql` / `aws` / `snow` / logs / metrics) goes through `big-think-scout` — the one role carrying the read-only, staging-by-default discipline, so parallel workers never hit prod. It also runs the reality probe at *every* posture's gate, not just diagnosis. A single scout task may weave across surfaces in one pass. A genuine *multi-source survey* is out of scope for a bounded run: the worker or scout flags it and you surface it to the user as a limitation, rather than diverting to another tool.
- **No change narration.** Every artifact describes the problem and the analysis as they currently stand. On a re-run, overwrite the file so it reads as written fresh — no before/after, no "updated to…".
- **Stay out of the documents.** Relay each artifact to the user by its path plus a 1–2 line summary; don't echo file bodies into chat. Your context is for orchestration and synthesis, not for holding whole documents.
- **Loop back when the evidence says the frame was wrong.** If the red team dismantles every candidate understanding, or the pre-mortem kills every approach, that is a signal the framing or the understanding was wrong — reopen the gate or the frame, don't patch forward. (The "if three fixes fail, stop — it's mis-framed" instinct, made structural.)
- **Cap the loops with hard limits.** Understanding: at most **2 fan-out rounds**. Gate: at most **2 red-team passes** and **3 scout probes** per leading candidate. Loop-backs (reopening the frame or the gate): at most **2** for the whole run. Convergence: **1 scoring round** plus the pre-mortem. When a cap is reached without convergence, stop and surface the impasse — an open-questions plan, or a decision under acknowledged uncertainty — rather than looping again.

## Run directory

Create a per-run workspace in the plugin's temp dir:

```
${TMPDIR:-/tmp}/code-goblin-pro/bigthink-<YYYY-MM-DD>-<slug>/
  problem.md          # the framed question, posture, scope/persona, evidence ledger
  hypotheses.md       # competing candidate understandings — scored, cross-linked, with evidence
  understanding.md    # the locked understanding + gate dossier   (OR open-questions.md if the gate fails)
  approaches.md       # candidate approaches, each mapped to what it addresses
  decision-record.md  # the deliverable: recommended approach + tradeoffs (ADR-shaped)
```

Name the run `<YYYY-MM-DD>-<slug>` — get the date with `date +%Y-%m-%d`, slug from the problem.

## The postures

big-think runs in **one** posture, chosen at triage. The posture parameterizes exactly four things — the first-diamond **target** (what "locked understanding" means), the **gate block** (Phase 2), the **lens set** (playbook below), and the **deliverable framing** — and nothing else. The second diamond (diverge on approaches → converge → decision record) is the same in every posture. The triage question is: **what is it you don't yet understand well enough to act?**

| Posture | Trigger | First-diamond target — the *locked understanding* | Defer instead to |
|---|---|---|---|
| **diagnose** | "why is this happening?" — an observed failure with an unknown cause (bug, regression, outage, perf cliff) | a **root cause** | a long interactive debugging dialogue → grillmaster |
| **frame** | "I must invent an approach / I don't know how to even start / this seems intractable" | a **precise problem spec + tractability/crux analysis** | turning the chosen approach into a buildable implementation spec |
| **orient** | "I've been handed a problem in a domain I don't know" | a **validated mental model + operational map** of the domain | a cited report on the domain → deep-research |
| **survey** | "how is this solved with current best-practice / SotA?" | a **bounded, decision-serving landscape** of approach families + fit criteria | a comprehensive cited report → deep-research |
| **decide** | "I have several options; which is right?" (build-vs-buy, A/B/C, migrate-or-not) | a **validated decision frame** — real option set · weighted criteria · what actually differs | inventing the options — that is `frame`, not `decide` |

One posture per run. If, mid-run, the real blocker turns out to be a *different* posture's understanding (you set out to `diagnose` but can't even map the domain), that is a **bounded loop-back** (Phase 0), not a planned pipeline — re-triage, run the prerequisite posture to its locked understanding, fold it in, and resume the original posture. A problem that obviously needs several postures up front is too big for one big-think run; surface that to the user rather than absorbing it.

**Two boundaries to hold in the skill's own judgment.** `survey` produces a *bounded landscape in service of a decision* — if what the user wants is the cited *report* itself, or comprehensive citation coverage, decline and name deep-research. `frame` stops at a recommended approach and its rationale — if the user wants an implementation-ready spec to build from, that is a separate spec-building step. big-think is for **understanding-risk, not planning-risk.**

## The lens playbook

Phase 1 fans out `big-think-analyst` workers, each dispatched with **one lens** — the analytical method it applies. You choose which lenses fire and how many from the posture and the problem's specifics. Each lens generates candidate elements of the understanding *independently* (workers don't see each other) and returns structured findings. Fire the lenses that fit, not all of them; typically two to four. The lenses are grouped by posture, with a couple shared.

**Shared** (available to any posture that needs them):
- **first-principles** — strip the problem to fundamentals and ask whether an assumed constraint is real; surface what the obvious framing hides.
- **analogy / transfer** — what known problem or domain this resembles, and *where the analogy breaks* (the break-point must be named, or the analogy misleads).

**diagnose** — the deterministic root-cause path:
- **historian** *(almost always)* — "what changed?" Reconstruct the timeline — deploys, merges, config/flag flips, dependency bumps, data/traffic shifts, infra events — against when the symptom began. Software's highest-yield first question.
- **boundary** *(concrete failing system)* — the seams: inputs, API/version contracts, assumptions that hold on one side of an interface and not the other, error propagation across components.
- **kepner-tregoe** *(concrete failing system with a sharp symptom)* — the IS / IS-NOT specification (What / Where / When / Extent): what exhibits the problem versus what could but doesn't. The true cause must explain every IS *and* every IS-NOT; candidates that contradict an IS-NOT are eliminated.
- **fault-tree** *(failure with several contributing conditions)* — take the symptom as the top event and decompose through AND/OR gates into the basic conditions that could produce it; find the minimal combinations.
- **fishbone** *(messy or multi-factor)* — sweep candidate causes across categories (code / config / data / dependencies / infra / process, or the domain's real categories) so no whole class is missed.
- **systems** *(feedback-driven or non-software)* — map the stocks / flows / feedback loops and name the dominant loop driving the symptom.

**frame** — pin down and locate the difficulty of a blank-page or seemingly-intractable problem:
- **formalize** — state the problem in precise terms; force the fuzzy parts to resolve; write the acceptance test that would recognize a correct solution.
- **constraint-map** — enumerate and classify every constraint (hard vs. soft); find which one is actually binding.
- **reduction** — is this a known problem in disguise? map it onto canonical problems.
- **relaxation** — which constraint, if dropped, makes it tractable — and is that relaxation acceptable?
- **invert** — what would make it *definitely* impossible? (locates the essential barrier).
- **decompose** — does it factor into independent sub-problems, some already solved?

**orient** — build an operational map of an unfamiliar domain:
- **terrain** — map the core entities, boundaries, and data/control flow of the domain.
- **forces** — the constraints, incentives, and failure pressures that shaped the domain; why it is the way it is.
- **exemplars** — the canonical worked examples / reference implementations that teach the domain fastest.
- **vocabulary** — the terms of art and what they precisely mean (misunderstood vocabulary is the top newcomer trap).

**survey** — map the solution landscape for a fit decision:
- **industry-standard** — what the mainstream / production world does.
- **state-of-the-art** — what the research frontier / leading edge does.
- **adjacent-field** — how a neighboring domain solves the structurally-same problem (often the highest-value transfer).
- **failure-museum** — approaches tried and abandoned, and *why* (keeps the survey from re-recommending a known dead end).

**decide** — frame a choice among largely-known options:
- **option-expansion** — surface unnamed options and the do-nothing / status-quo baseline.
- **criteria-elicitation** — what actually matters here, and how much.
- **steel-man** — argue the strongest case *for each* option independently (prevents anchored dismissal).
- **stakeholder** — whose interests and constraints bear on the choice, and where they diverge.
- **reversibility** — for each option, the cost and difficulty of undoing it.

**When the first diamond can't be analyzed, probe.** On the **diagnose** posture a **Complex** problem (Cynefin) replaces analysis-to-a-cause with **safe-to-fail probes**: instead of the lens fan-out, dispatch the Scout to run **at most three or four** small, read-only, reversible experiments — each perturbing or observing one suspected factor — and record what each reveals in `hypotheses.md`. `frame` and `orient` carry the same instinct in lighter form — when the understanding can't be reasoned out (is this tractable? does this domain behave as I think?), attempt a reduction, recommend a tiny prototype, or run an exploratory probe rather than asserting. In every case, probe findings become a gated understanding only if one signal is strong enough to state a mechanism and a falsifiable prediction; if the signal stays diffuse, the honest output is `open-questions.md`, never a forced understanding.

## Phase 0 — Triage, select posture & frame

**If big-think was invoked with no problem stated, ask the user for it first — never invent one.** Otherwise create the run directory and write `problem.md` from the template below, filling it by provenance: what the user stated, what you verified from a source (tagged), what you are assuming (flagged, never baked into a field), what is still open. Establish scope and persona explicitly — do not silently import them. (Exception: if a quick first read shows this is a **Clear**, already-answered problem — a known cause, a known domain, a known best-practice, an obvious choice — skip the scaffolding entirely: give the known answer or runbook and stop, per below.)

Then do these things:

1. **Select the posture** — the primary dial (see the postures table). Ask *"what is it you don't yet understand well enough to act?"* and pick exactly one: **diagnose / frame / orient / survey / decide**. The user's own words usually decide it. Record it in `problem.md`. If genuinely torn between two, that ambiguity is the thing to resolve at the intake STOP — don't run two machines.
2. **Apply the Clear / Chaotic filter** (global, every posture):
   - **Clear** (the understanding already exists) → give the known answer or runbook directly and stop — don't create a run directory or spin up the machine.
   - **Chaotic** (active crisis, no stable ground to analyze) → don't analyze first. Recommend the single stabilizing / containment action that calms the system, write it into `decision-record.md` as an immediate action, and note that once stable the problem is re-triaged. Containment precedes understanding. (Mostly relevant to diagnose; occasionally decide under time pressure.)

   For the **diagnose** posture *only*, additionally sub-classify **Complicated vs. Complex** — Complicated dials deterministic analysis-to-a-cause via the lens fan-out; Complex dials safe-to-fail probes (see the playbook). The other postures don't use Cynefin's method dial; they use their own lens sets, with the lighter "analyzable vs. must-probe" choice noted above.
3. **Check the frame** (cheap, one pass): is this the right problem? Whose problem is it, what is the gap between desired and perceived, is there a better problem to solve? A quick inversion — "what would guarantee this stays unsolved?" — often exposes the real target. Reframe before diverging if the frame is off.
4. **Ground the question if needed** — dispatch `big-think-scout` once when the reported problem should be confirmed against reality (a live dashboard, a log, "is this already known in the org?", "does this domain actually work as stated?") before you trust the framing.

**STOP — only if the posture, framing, or scope is genuinely ambiguous.** Present `problem.md`, resolve the ambiguity, and fold the answers in. If the posture, problem, and scope are already clear, do not stop — proceed to Phase 1.

## Phase 1 — Diverge on understanding

Draft the structure in `hypotheses.md`: a MECE-ish map of the space the understanding must cover, so the fan-out covers it without overlap or gaps. (For diagnose this is the issue tree of where the cause could live; for frame the facets of the problem to pin down; for orient the regions of the domain to map; for survey the space of approach families; for decide the option-and-criteria space.)

Fan out `big-think-analyst` workers — one per chosen lens (see the playbook; count gated by posture and problem specifics, typically two to four). Each dispatch prompt carries the run directory path, the framed problem inline, the **posture**, the **lens and its method**, the **slice of the space** it owns, and the output contract (structured findings: candidate element of the understanding · why/how it holds · evidence-for · evidence-against · confidence · the discriminating test that would confirm or refute it). Workers read the local tree directly and **name off-tree questions in their return** rather than reaching into org repos or live systems themselves — you route those to the Scout so five lenses don't hit prod in parallel.

Merge the returned findings into `hypotheses.md` as first-class, scored, cross-linked entries. **Dedup by *substance*, not by wording** — collapse two lenses describing the same thing into one (noting the cross-lens agreement, which raises confidence), but keep genuinely distinct rivals separate even when they point the same way. Where one finding is a component or symptom of another, record the parent/child chain explicitly rather than merging them. This is the "Discover" diverge; you converge it at the gate.

## Phase 2 — Understanding Gate  *(hard gate)*

Rank the candidate understandings by evidence and discriminating power (prefer the next check that best *separates* the leaders; rule out anything a probe or an IS-NOT contradicts; keep a "can't-miss" set of high-severity candidates explicitly excluded, not silently dropped). Run the cheap discriminating tests — dispatch the Scout to execute the **read-only probe against the leading understanding's load-bearing assumption** (query the DB, pull the metric/log window, inspect deployed config, verify the baseline is actually standard, confirm the constraint is real, check the decomposition hasn't already been solved elsewhere). **The gate is a probe against reality, not just an argument — at every posture.**

Then attack the leader: dispatch `big-think-red-team` (one or two independent passes) with the posture's attack fronts, to break the leading understanding and name the decisive outstanding test.

**Exit criteria — the leading understanding passes the gate only if ALL hold.**

*Universal skeleton (every posture):*
- **Grounded in reality** — a read-only probe was run against its load-bearing assumption, and the result is consistent with it.
- **Constrains · Predicts · Directs** — it rules out some tempting solution paths, predicts something checkable (and the check held), and names what any viable approach must address.
- **Evidence for and against** — both recorded; the against side doesn't sink it.
- **Survived the red team** — with a stated confidence level.

*Posture block (additive):*
- **diagnose** — **Mechanism** (a causal chain from cause to symptom, not a correlation) · **Explains affected *and* unaffected** (the IS / IS-NOT test) · **Falsifiable** test whose outcome would disprove it, with an actual result consistent with the cause · the four-property **root-cause test**: Actionable · Preventable · Fundamental · Verifiable.
- **frame** — **Well-posed** (you could recognize a correct solution — the acceptance test is stated) · **Invariants explicit** (hard constraints enumerated, soft ones marked as preferences) · **Difficulty located** (the *essential* source of difficulty named and separated from accidental complexity) · **Tractability assessed** (tractable / tractable-under-relaxation / likely-intractable-as-stated, grounded in a mechanism or known result — if intractable, the specific constraint to relax is named) · **Reduction checked** (isn't this a known-solved problem in disguise?).
- **orient** — **Predictive** (the model made a concrete prediction that was checked against a real source and held) · **Grounded not hallucinated** (every load-bearing claim traces to code / doc / observed behavior, not domain-priors) · **Boundaries marked** (what it does not cover, and where confidence drops) · **Actionable leverage** (at least one concrete next move, and why it is the leverage point) · **Survives the "an expert would wince" test** (red-teamed for the newcomer misconception).
- **survey** — **Exhaustive-enough** (no dominant approach family missing) · **Discriminated not listed** (families separated by the dimension that actually distinguishes them; two that collapse to the same tradeoff are merged) · **Current & sourced** (each family grounded in a real source, recency noted) · **Fit criteria derived** from *this* problem's constraints, stated *before* any family is favored · **Maturity-honest** (battle-tested vs. bleeding-edge marked per family).
- **decide** — **Option set complete** (the strong options *and* the do-nothing / status-quo baseline; red-teamed for the missing option) · **Criteria grounded & weighted** *before* scoring, not reverse-engineered to justify a favorite · **Real differences isolated** (where options genuinely diverge, separated from where they're a wash) · **Reversibility & stakes surfaced** (one-way vs. two-way door made explicit) · **No premature favorite** (the frame is judged before any winner is named).

- **If it passes** → write `understanding.md` (the locked understanding + the dossier above).
- **If nothing passes** → write `open-questions.md`: the surviving candidates and the specific discriminating tests / probes to run next to tell them apart, or the missing access / data / decision that blocks a confident understanding. This is a legitimate, honest terminal outcome — **not** a fake answer. Per posture: diagnose → an investigation plan; frame → "intractable as stated / reduces to X — the constraint to relax"; orient → the unresolved unknowns and how to close them; survey → "the landscape is incomplete here — needs deep-research"; decide → "the frame can't distinguish these — you need info on X first."

On the **diagnose Complex path**, the "understanding" is the pattern the probes revealed, and its falsifiable test is a further probe whose outcome it predicts; if no pattern is strong enough to meet every criterion, exit to `open-questions.md` as an experiment plan rather than weakening the criteria to force a pass.

**STOP (required).** Present `understanding.md` (or `open-questions.md`). A plain **confirm** — or a wording/scope tweak that leaves the understanding intact — advances to Phase 3. A **correction that changes the understanding**, or a replacement the user proposes, is *not yet gated*: re-enter the gate for it (re-probe and re-red-team against the exit criteria) before any solutioning — a user's say-so is not a substitute for surviving the gate. Nothing proceeds to approaches without a *gated* understanding. If the run ended in open questions, this is where it ends.

## Phase 3 — Diverge on approaches

*(Reachable only through a confirmed gate.)* Reframe the locked understanding as one or more **How-Might-We** questions. Fan out `big-think-idea` workers — one per **stance** (typically minimal-intervention · structural/root-fix · lateral/reframe) — each dispatched with the locked understanding inline, the run directory path, its stance, and the constraint that **every proposed approach names what it addresses** (the causal link / constraint / crux / criterion in `understanding.md`) and honors the problem's scope and constraints. Generation only — the idea agents don't rank or pick. *(For the **decide** posture the options are already fixed by the frame — dispatch the idea workers to steel-man each option independently rather than invent new ones.)*

Compile the returned approaches into `approaches.md` as comparable profiles: approach · what it addresses · cost/complexity · risks & reversibility · what it deliberately doesn't do.

## Phase 4 — Converge & record

Score the surviving approaches against weighted criteria — does it address the understanding · cost/effort · risk · reversibility · blast radius — a simple decision matrix; where it's close, weigh the criteria deliberately rather than by first impression. Before locking the leader, two cheap adversarial gates:

- dispatch `big-think-premortem`: "this approach shipped and the problem came back, or a new one appeared — why?" Fold the top failure modes into the risks.
- run a **second-order** check yourself: "and then what?" — does a later-order effect reverse the benefit?

Converge on **one** recommended approach (or a tight ranked pair if it is genuinely too close to call), and write `decision-record.md` (template below): the problem, the locked understanding and its confidence, the recommended approach and why, the alternatives considered and why they lost, the key risks (including the pre-mortem findings and any second-order effects), and the open questions. For a software problem it is ADR-shaped; for a non-software problem it is a plain decision record. **It recommends an approach; it does not prescribe running another tool.**

**Loop-back:** if the pre-mortem kills every approach, or scoring shows nothing actually addresses the understanding, the understanding or the frame was wrong — reopen the gate (Phase 2) or the frame (Phase 0). Don't ship an approach that doesn't address the understanding.

**STOP (required).** Present `decision-record.md` by path plus summary, surfacing the recommendation and its tradeoffs for the user's decision.

## Templates

### problem.md

```markdown
# big-think · <problem slug>

## The question
What you don't yet understand well enough to act on — the observable, concrete problem, not a guess at the answer. (For diagnose: the symptom, concrete and specific.)

## Posture
diagnose | frame | orient | survey | decide — and one line of why. For diagnose, add the Cynefin sub-class (Complicated | Complex) and note any Clear/Chaotic handling. This dials the run.

## Scope & persona
Whose problem this is, what system/area is in scope, what is explicitly out. (User-stated; inferences flagged.)

## Desired vs. perceived
What "understood / resolved" looks like, and the gap between the desired and the current state.

## Evidence ledger
What we know, each tagged: (stated) / (verified: <path-or-url>) / (assumed — confirm) / (open).
- ...

## Constraints
Hard limits on any approach — compatibility, don't-touch-X, budget, deadline. User-stated.

## Assumptions to confirm
Inferences made that were not stated or verified — one line each, for the user to confirm / drop / correct.

## Open questions
Genuinely undecided; these feed the first diamond.
```

### hypotheses.md

```markdown
# Candidate understanding · <problem slug>

## Map of the space
What the understanding must cover (MECE-ish), so the search covers the space.
- ...

## Candidates

### H1 — <one-line candidate>
- **Lens(es):** which lens(es) surfaced it; note cross-lens agreement
- **What it says:** the claim / mechanism / model element
- **Evidence for:** … (tagged with source)
- **Evidence against / doesn't explain:** …
- **Discriminating test:** the check that would most confirm or refute it
- **Confidence:** low | medium | high — on what basis

### H2 — …
```

### understanding.md

```markdown
# Locked understanding · <problem slug>

## Locked understanding
<one-paragraph statement>. (For diagnose, this is the root cause.)

## Dossier (gate criteria)
- **Grounded in reality:** the probe run against the load-bearing assumption, and its result.
- **Constrains / Predicts / Directs:** what it rules out · what it predicts (and the check) · what any approach must address.
- **Evidence for / against, and why the against doesn't sink it:** …
- **Survived the red team:** confidence level, and how the red team was answered.
- **Posture criteria:** each item of the posture block (diagnose / frame / orient / survey / decide) — satisfied how.

## Ruled out
The rejected candidates and the evidence or probe that eliminated each.
```

### open-questions.md  *(when the gate does not pass)*

```markdown
# Open questions · <problem slug>

_The gate did not pass: no candidate met the exit criteria. This is the plan to get there — not an answer._

## Surviving candidates
The candidates still in contention, with current confidence.

## Discriminating tests / probes to run
Each: the test/probe · which candidates it separates · what result implicates which · cost/access needed.
1. ...

## What blocks a confident understanding now
Missing access, data, or a decision — and what would unblock it.
```

### decision-record.md

```markdown
# Decision record · <problem slug>
_Status: recommended — for your decision_

## Context
The problem, and the **locked understanding** (confidence: …). See `understanding.md`.

## Recommended approach
What to do, and **what it addresses**. Why this one over the alternatives.

## Alternatives considered
Each approach weighed, and why it lost — didn't address the understanding / cost / risk / reversibility.

## Consequences & risks
Positive and negative outcomes; the pre-mortem's top failure modes and any second-order effects; what this deliberately does not do.

## Open questions
What remains unresolved or needs the user's call before acting.
```
