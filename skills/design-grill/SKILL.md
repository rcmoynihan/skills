---
name: design-grill
description: Grill the technical side of a compiled spec relentlessly — architecture, data/state, interface realization, execution, ops, deployment — one question at a time, recommending an answer for each, while keeping a durable on-disk agenda. Holds every element of the spec as locked input; resolves high-level design decisions, not an implementation plan. The third link in spec-grill → to-spec → design-grill → to-design.
argument-hint: "(optional) path to a grill run dir or spec file; blank uses the most recent run"
disable-model-invocation: true
---

# Design-Grill

Grilling a design is a relentless, one-question-at-a-time interview: walk every branch of the technical decision tree, resolve decisions one by one, and recommend an answer for each question. This grill covers **the technical side only** — the *how*: architecture, data and state, interface realization, execution, reliability, security, operations, deployment. The product *what* was settled by `/spec-grill` and compiled by `/to-spec` into the spec this grill holds as **locked input**. The failure mode is **drift** — when the user corrects you or opens a side topic, the latest turn dominates, you rabbit-hole, and you never climb back up to the broader agenda. Design-grill keeps the interview but gives it a durable map so it can **follow tangents without being trapped by them.**

**You are the interviewer and the orchestrator.** You ask the questions, maintain a cheap on-disk map of the exploration as you go, and dispatch four subagents. Three carry the heavy reasoning: the **Planner** (`grill-planner`) decomposes the spec's technical surface up front, the **Facilitator** (`grill-facilitator`) periodically pulls you back up to the agenda, and the **Tracer** (`grill-tracer`) runs concrete operational scenarios through your resolved decisions to catch the contradictions that surface only on execution. The fourth is the **Scout** (`grill-scout`) — an on-demand researcher/verifier you send out whenever a question turns on a fact you don't yet have (how a subsystem actually works, what exists org-wide, how a live service behaves, what a third-party API guarantees, an established best practice), so answers rest on reality instead of guesses.

One thing to be honest about with yourself: you are both the thing that drifts *and* the thing keeping the map, so the map you write can itself be drift-colored. On-disk state cleanly recovers the thread **across a context reset** (compaction, resume). Within a single session it is the two independent readers your drift hasn't touched that keep you honest, aimed at different targets: the **Facilitator** catches *process* drift — circling, rabbit-holing, branches left unpopped, invariant erosion — by reading the map; the **Tracer** catches *design unsoundness* — decisions that read as consistent but collide on a concrete run — by executing it. Lean on both, and keep your own bookkeeping lean so there's less for a drifting interviewer to mis-set.

## The lane boundary — technical only, between two walls

This grill resolves technical decisions and nothing else. It is bounded on both sides:

- **The product wall (below is locked).** The test is the same one `/to-spec` and `/to-design` use to route content: **would this change if we re-implemented the same product a different way?** No → it's product — already decided by the spec, **locked, not discussed**. A user pull toward re-deciding a product question is not a tangent; it's a **spec-amendment escalation** (see below), the only door through this wall. What the spec left genuinely open (its Open Questions) stays open — you never resolve a product question in this lane.
- **The implementation floor (below is out of scope).** This is high-level design, not an implementation plan. Resolution bottoms out at design-level mechanisms — a component and its responsibility, a schema shape, a procedure, a consistency model, a rollout strategy — never file paths, class/function names, package versions, or task sequencing. (This mirrors `/to-design`'s no-volatile-detail rule: what this grill resolves is exactly what the design doc can print.) When an answer starts naming files and functions, lift it back to the mechanism it instantiates.

Between the walls, everything technical is in scope at the posture's depth: architecture and component boundaries; domain model, data, and state; the *realization* of the spec's interface promises (schemas, error codes, idempotency, versioning); execution and data flow; the *mechanisms* that meet the spec's NFR targets; failure modes and recovery; security and privacy; observability; deployment, rollout, and rollback; test and verification seams.

## How to run this

- **One question at a time.** Asking several at once is bewildering. Recommend your best answer for every question you ask. If a question is answerable by a quick local read of the codebase or a referenced doc, go read it inline instead of asking. If it turns on something heavier — how a subsystem actually works, what exists org-wide, how a live service behaves, what a third-party API guarantees, an established best practice — **dispatch the Scout** to find out, rather than guessing or handing the user your homework.
- **Dispatch four agents, by name.** `grill-planner` (once, at the start), `grill-facilitator` (periodically), `grill-tracer` (when a cluster of operational decisions closes, and at the pre-completion gate — see *Dispatching the Tracer*), and `grill-scout` (ad-hoc, whenever a decision needs external grounding — see *Dispatching the Scout*). Use the Agent tool (`subagent_type`); if `/agents` shows them plugin-scoped, use that form. **Every dispatch names the lane and the lane dir:** these are shared workers serving both grills, so each prompt carries `lane: design`, the lane dir path, the posture, and the spec path (`grill-<slug>/spec.md`) — subagents can't see this conversation. The Planner also spawns its own `grill-scout` child at scaffold time to ground the agenda in the codebase this design lands in. Everything else — the per-turn log, the givens ledger, the status updates, the recovery bookkeeping — you do inline. Never dispatch an agent per turn; a per-turn logger would just be re-fed what you already hold. The Scout and the Tracer fetch or test what you *can't* settle yourself — dispatch them on their triggers, not routinely.
- **You are the sole writer of the three living files** (`living-agenda.md`, `conversation-path.md`, `givens.md`). The Planner writes `initial-agenda.md` once and only *reads* the givens ledger — it reports each given's route and you transcribe it; the Facilitator and the Tracer write nothing — they return exact lines you transcribe. One writer per file, no races. The living agenda's **Locked block** has its own stricter rule below.
- **Restructuring the agenda is not your unilateral call.** Routine changes stay inline — adding a single tangent/new-concern node under an existing area, flipping a status, dropping a stray tangent that fizzled. But *substantial* restructuring — dropping or re-scoping a standing branch, merging or splitting nodes, adding a whole new branch or area — routes through the Facilitator: you propose it (with the live rationale it can't see), it rules, and you apply its verdict by **transcribing the exact changes it returns**, verbatim. You are the pen; the restructuring decision belongs to the un-drifted reader, precisely because you are the one that drifts.
- **Keep the per-turn cost near zero.** Append one row to the log (never rewrite it); rewrite only the small Position block; flip a status token; append a givens line (plus its node annotation) when one lands; flip a given to `[consumed]` when its node resolves with it folded in. Write the next log row from what's in your context plus the Position block — **almost never re-read the full log.** Only the Facilitator reads the whole log.
- **No change narration in the agendas.** The agendas describe the exploration as it currently stands: status is a field, never "was active, now paused". A newly added concern carries a *standing rationale*, never an event. The one exemption is `conversation-path.md`, which is a log: it narrates the *conversation* but never narrates *artifact edits*.

## Posture — inherited, not proposed

The run's rigor tier — `poc` / `internal-tool` / `product-feature` / `new-system` — was proposed, confirmed, and pressure-tested in the spec grill; `/to-spec` recorded it in the spec's frontmatter. This grill **inherits** it: the Planner reads it from `spec.md` and records it as inherited; you state it in your opening move ("correct me if the tier changed") but never re-propose it. It lives in the Position block as `posture: <rung> — <optional note>` and dials depth exactly as before: how hard each item is pressed, and when an item is *resolved enough* — a `poc` design closes on the core mechanism; a `new-system` design isn't done until security, ops, scale, and migration have been pressed.

If the user *upgrades* the tier mid-run, re-dispatch the Planner to extend the agenda additively (a heavier tier pulls in areas the map may lack — security, ops, scale, migration); a *downgrade* keeps the whole map and dials depth down — never prune. Either correction also means the spec frontmatter's posture is stale: record it for the completion summary, which instructs re-running `/to-spec`.

## Inputs, run directory & resuming

This grill runs inside the run dir the chain already created; it never mints its own slug. Discover it:

```bash
ls -dt "${TMPDIR:-/tmp}"/code-goblin-pro/grill-*/ 2>/dev/null
```

If the argument names a run dir or a spec file, use that; otherwise take the most recent run matching the topic. Place the run in the chain:

- **`spec.md` is the required input.** No `spec.md` in any matching run → **decline and route**: `/spec-grill` to interview the idea first, or `/to-spec` to compile a spec from an existing conversation. A design grill with nothing to lock has no ground to stand on — there is no standalone mode.
- **Enrichment, when present:** `spec/technical-parking-lot.md` (the deflected technical asides — the Planner's seed material) and the spec lane's state files (decision provenance). Parking-lot lines carrying a `[decided|leaning]` conviction tag — or, in a spec-only run, tagged §12 Carried-Forward lines — are copied into this lane's `givens.md` at seed time (origin `spec-lot`): the user's stated technical convictions, held as givens rather than open candidates. A run dir holding only `spec.md` — a conversation-born spec with no spec lane — is valid; enrichment is *when present*, not required.
- `design/living-agenda.md` already exists — a design grill in flight. Offer to resume: read the lane's files, rebuild your position from the Position block, and continue the interview; `givens.md`'s `[open]` entries are standing details to re-hold (`decided` held as the user's call, `leaning` as the default recommendation for its node).
- `design.md` exists — this lane is already compiled. Starting a new design grill here is a re-grill: warn that `design.md` goes stale and confirm before proceeding.

This lane's state lives in `grill-<slug>/design/`:

```
grill-<slug>/
  spec/ …                      # the spec lane (read-only to this grill)
  spec.md                      # the locked input (read-only to this grill)
  design/                      # this grill's lane
    initial-agenda.md          # frozen first map (Planner writes once)
    living-agenda.md           # the working map (you own it)
    conversation-path.md       # append-only turn log (you own it)
    givens.md                  # a-priori details the user brought (you own it)
  design.md                    # written later by /to-design
```

No parking lot in this lane: a product pull escalates (see *Spec-amendment escalation*), it doesn't park — there is no later lane to park into. A-priori *technical* details go to this lane's `givens.md`; a pull to re-decide product content is never a given — it takes the escalation.

## The four state files

**`initial-agenda.md`** — the frozen first map, written once by the Planner. At its top sits the **Locked block**; below it, lettered concern areas and numbered `[unvisited]` items. Read it to seed the living agenda and again only at completion (the coverage diff). **Don't re-read it mid-grill.**

**`living-agenda.md`** — the working map, three parts top to bottom:

- **Position block** — rewritten wholesale each turn: `posture` (the inherited tier), `active`, `resume-target`, `turns-since-node-change`, `last-facilitator`.
- **Locked block** — copied once from `initial-agenda.md` at seed time, sitting *between* the Position block and the agenda body, deliberately outside the wholesale rewrite: locked text must never pass through a drift-colored rewrite. It opens with the standing rule — *"The entire spec.md is locked input; these are its sharpest edges. No agenda item may resolve against them."* — followed by every spec `INV-` and binding `CON-` id with its one-line text **verbatim**. It is edited in exactly one case: a user-confirmed spec amendment updates the affected line in place, tagged `(amended)` (see *Spec-amendment escalation*).
- **Agenda body** — each item a status token + ID + question, statuses **`unvisited` / `active` / `paused` / `resolved` / `dropped`** with the same depth tags as the spec grill: `[resolved: mechanism]` (names a concrete artifact/state/procedure), `[resolved: firm-up]` (an approximation fine to make precise later in /to-design — e.g. "~2–3 retry attempts"), `[resolved: deferred]` (explicitly punted, or an accepted risk). One extra pause tag is this lane's own: `[paused: blocked-on-spec <spec OQ/section tag>]` — the item can't be designed until a product question the spec left open is answered, and that answer belongs to the spec, not to you (see *Drift recovery*). New items get a trailing `— reason added: <standing rationale>`; an item seeded by a given carries `— given G<n> (decided|leaning): <one line>`.

**`conversation-path.md`** — the append-only turn log, one row per turn, fixed columns:

```
turn | active-node | move | summary | implication | drift | next-node
```

`move` ∈ `answer | correct | tangent | new-concern | amend | given | meta`. `correct` and `tangent` are distinguished exactly as in the spec grill — their recoveries differ. `amend` is this lane's own move: a locked product decision was changed through the escalation below — the row names the spec id and the new decision. `given` marks a turn that was *wholly* an a-priori technical detail aimed at a node that isn't active — you appended a givens line, annotated the target node, and stayed put; a given riding along with a substantive answer keeps the turn's real move, with the filing noted in `implication`. A pull to re-decide *product* content is never a `given` — that's the escalation. `meta` covers a non-user event that changed your read of the map — a Facilitator pass, a Tracer pass, or a Scout finding that materially informed a decision — so a node closed on a *verified* fact is legible as such. `summary` and `implication` are ≤12 words each. `drift` ∈ `none | watch | deep`.

**`givens.md`** — the ledger of a-priori details the user brought, at intake, mid-grill, or carried in from the spec lane: one line per given, `- [open] G1 (decided, intake) <one-line detail> → B3` — status, stable ID, conviction + origin (`intake`, `turn N`, or `spec-lot`), the detail, and the node it routes to (`→ —` until routed). Statuses: **`open` / `consumed` / `superseded` / `set-aside`**. Conviction: `decided` — the user's settled call, held once confirmed; `leaning` — a preference, the default recommendation for its node. Entries are append-only; only the status token and the route mutate — current state lives in the fields, never in appended notes.

## Spec-amendment escalation

The spec is locked, but the user owns it — sometimes designing reveals the product decision itself was wrong. When the user's answer, or a Tracer/Facilitator finding, contradicts a Locked entry or any spec statement, **never silently absorb it**: name the specific spec id or section out loud, then present the fork with your recommended side:

- **(a) Keep the spec** — treat the constraint as firm and re-answer the design question inside it.
- **(b) Amend the spec** — the user states the new product decision, in product terms.

On amend: update the affected Locked-block line in place to the new constraint, tagged `(amended)` — the tag means "this entry supersedes `spec.md`", and downstream tooling keys on it. Log the turn as `move: amend` (spec id + new decision in `summary`). Continue the interview under the amended constraint.

State the consequence plainly when it happens: **`spec.md` is now stale.** The completion summary lists every amendment, and the chain rule is fixed: **re-run `/to-spec` before `/to-design`** whenever any `(amended)` entry exists — `/to-design` refuses to compile against a stale spec.

A `decided` given is not a Locked entry: it is the user's own standing instruction in this lane, so overturning one is an ordinary `correct` (flip its ledger line to `[superseded]`) — no escalation, no spec staleness. The escalation owns exactly one given case: a given that contradicts a Locked entry (the Planner flags these at routing) forces this fork when its node goes active — the given is evidence for side (b), never a silent override.

## Phase 1 — Scaffold the agenda (Planner)

Create `grill-<slug>/design/`, then run **intake**: create `givens.md` with just its header; extract every a-priori technical detail embedded in the user's invocation (origin `intake`); copy each conviction-tagged parking-lot line — or, in a spec-only run, tagged §12 Carried-Forward line — as a given (origin `spec-lot`, conviction preserved). Tag `decided` only when the user stated it as settled ("must", "we're using X"); otherwise `leaning` — when unsure, `leaning`. Dispatch `grill-planner` with `lane: design`, the lane dir path, the spec path, the givens path when any were filed, and the parking-lot / spec-lane paths where they exist (plus any codebase paths the user named). The Planner reads the spec in full, records the **inherited posture** and writes the **Locked block** at the top of `initial-agenda.md`, grounds itself in the codebase via its own `grill-scout` child, then decomposes the technical surface into concern areas that **mirror the `/to-design` template's section spine** (architecture; data & state; interfaces; execution; reliability; security; observability; deployment; test seams — the areas the spec's domain actually has), seeding items from the spec's technical Open Questions, each NFR target (what mechanism meets it), each interface promise (what realizes it), the parking-lot lines, and what the codebase already provides — threading each given into the item it informs. When the Planner returns, transcribe the routes it reports into the ledger's route fields.

Seed the living agenda from it: fresh Position block (`posture:` the inherited tier, `active: —`, `resume-target: —`, `turns-since-node-change: 0`, `last-facilitator: —`), then the **Locked block copied verbatim**, then the concern areas and items (given annotations ride along), every item `[unvisited]`. The living copy must read as clean current state.

Give the user your **opening move** — not a posture proposal: state the inherited posture ("correct me if the tier changed"), the locked baseline (n invariants held from the spec — the spec is locked input), the spec's unresolved product questions you're carrying as known holes, the givens you're holding (each with ID, conviction, and route — acknowledgment, not resolution: each `decided` given is confirmed when its node goes active), and a short summary of the agenda (the concern areas, where you'll start). Extend the standing invite: anything the user already knows they want, said now or any time, gets filed. Then start interviewing — no blocking confirmation gate.

## The interview loop

Each turn, in order:

1. **Ask** the one next question, with your recommended answer. Wait for the user.
2. **Sort the reply against the lane boundary** — a pull to re-decide product content triggers the spec-amendment escalation; an answer sinking below the implementation floor gets lifted back to its mechanism.
3. **Log** — append one row to `conversation-path.md`: the active node, the `move`, a ≤12-word summary and implication, the `drift` read, and where you're headed next.
4. **Update the map** — flip the active item's status if it resolved (**at most one standing item per turn** — closing several at once is how a shallow answer slips through unexamined), and flip its given to `[consumed]` when the resolution folded one in; add any new concern; rewrite the Position block (bump or reset `turns-since-node-change`, update `active` / `resume-target`). The Locked block is untouched except on `amend`.
5. **Check the triggers** (cheap reasoning, no tool call — see below). If a hard trigger is due, dispatch the Facilitator before asking the next question.

### Dispatching the Scout

When forming your recommended answer, if the honest answer is *"I'm guessing"* and the guess is load-bearing, send the Scout out instead. Dispatch `grill-scout` (with `lane: design` and the lane dir) when a decision turns on how a subsystem actually works, what already exists (in this repo or org-wide), how a deployed service behaves, what a third-party API guarantees, or an established best practice — and it's more than a quick local read you'd do inline. Also send it to **verify a factual claim** — yours or the user's — before it gets baked into a resolved item.

Pass it the **one specific question or claim** and the **scope** to look in: which repo/paths, which service, which vendor's docs. It reaches all five: the local codebase, the org-wide codebase (via `gh`), deployed/live services (read-only staging by default), third-party docs, and the web. It returns a sourced answer with a confidence read and what it couldn't confirm; it reports facts and does not decide the design — that stays with the interview.

Fold the finding into your recommended answer for the node. When it materially informed a decision, log a `meta` row (finding + source) so a verified closure reads differently from an assumed one. Keep it ad-hoc and lean — the Scout grounds a decision, it doesn't stall the interview.

### Drift recovery

- **`correct`** — re-answer the current node with the user's correction folded in. Stay on the node; don't pivot the whole interview around a single correction. If the "correction" actually contradicts a Locked entry, it's not a `correct` — run the spec-amendment escalation.
- **`tangent` / `new-concern`** — if the concern is a single new node under an existing area, add it inline (`reason added: …`); if it's a whole new branch or area, that's substantial restructuring — take it to the Facilitator, don't scaffold it yourself. Either way, set `resume-target` to the current node **only if it isn't already an outstanding resume obligation** — record the *shallowest* unfinished node. If `resume-target` is already set you're nesting: keep the shallow pointer — nesting is a hard Facilitator trigger. Mark the node you're leaving `[paused]`, make the tangent `[active]`, and descend.
- **`amend`** — the Locked-block edit, the log row, and the plain statement of spec staleness, per *Spec-amendment escalation*. Stay on the node and re-answer it under the amended constraint.
- **`given`** — the map doesn't move: append the givens line, annotate the target node, stay on the node, re-ask the question you were on. A given with no matching node gets a single new node inline under the right area (`reason added: …`); with no home area, file it `→ —` and take the area to the Facilitator via the restructuring path. When a given-annotated node goes active: a `leaning` given is your recommended answer, grilled at the posture's normal depth; a `decided` given is confirm-once-and-hold — restate it, voice disagreement at most once, and close on the user's confirm, still bottoming out in a concrete mechanism (a label-shaped given holds the *choice* while you grill the mechanics beneath it). Once held, don't re-argue it; only a collision — a Tracer or Facilitator finding, or a later decision that contradicts it — reopens it, and the user adjudicates: keep the given and revise the colliding decision, or overturn it (a `correct` move; ledger line → `[superseded]` — never an `amend`).
- **Blocked on the spec** — when an item turns out to need a product decision the spec left open (one of its Open Questions, or a genuine hole), don't answer it here: mark the item `[paused: blocked-on-spec <spec tag>]` and move on. It surfaces in the completion summary as needing a spec round-trip; only the user amending or extending the spec unblocks it.
- **On resolution** — before you mark a node `[resolved]`, name the concrete mechanism its answer bottoms out in: the artifact it produces, the state it lives in, or the procedure that decides it. Tag the close `[resolved: mechanism]`, `[resolved: firm-up]`, or `[resolved: deferred]`. An answer that only *renames the hard part* with a concrete-sounding phrase — "commit straight to X", "tests arbitrate", "handle Y" — is not a resolution; keep it `[active]`. A load-bearing decision must reach `mechanism` to close; only genuine downstream precision may rest at `firm-up`. A stray tangent that fizzled you may `[dropped: <reason>]` inline; dropping or re-scoping a *standing* item routes through the Facilitator. Climb back to `resume-target`; if none, take the next `[unvisited]` item in agenda order. Clear `resume-target` when you've returned to it.
- Most tangents are *abandoned*, not resolved — the user wanders off and never closes them. You won't always notice; that's what the Facilitator is for.

### When to call the Facilitator

The Facilitator is your only un-drifted reader, so err toward calling it. The triggers split into hard ones (no judgment — dispatch) and soft ones.

**Hard triggers — always dispatch:**

- **Same-node staleness:** `turns-since-node-change ≥ 4` — you've circled one node without resolving it.
- **Cadence:** `≥ 4` turns since `last-facilitator` (count from the start if it never ran) — tighten to `≥ 3` if any turn since then was a `tangent`, `new-concern`, or `correct`.
- **Nesting:** a new `tangent`/`new-concern` opens while `resume-target` is *already set*.
- **Sticky tangent:** a tangent that has survived more than one user turn without resolving or being dropped.
- **Deep drift:** you logged `drift: deep` this turn.

**Soft heuristics — consider dispatching:** a branch that feels finished; genuine uncertainty about where to go next; a run of `correct` moves on one node; a given filed with no home node; you've run the amendment escalation more than once in a few turns — repeated pulls against the spec may mean the spec, not the design, is what needs work.

**When it's fine to skip — and only then:** `resume-target` is empty **and** the tangent closed within one turn **and** drift was `none`/`watch` **and** no hard trigger is due. "I handled it correctly inline" is **not** a skip condition.

**Restructuring the agenda — dispatch on demand:** whenever substantial restructuring seems warranted, dispatch the Facilitator with your **concrete proposal and its rationale**, regardless of the cadence. This is the *only* path to a substantial add/drop/re-scope. (The Facilitator may also *return* a restructuring recommendation unprompted on any ordinary pass.)

Dispatch `grill-facilitator` with `lane: design`, the lane dir path, and the spec path; for a **restructuring request**, also hand it your concrete proposal and the live rationale it can't see. It reads the Locked block as part of its input and hunts **invariant erosion** — a resolved item whose answer contradicts a Locked entry, or a product decision changed without an `amend` row — alongside the usual drift tells. Treat its verdict as **binding**: on a `drifting` verdict, make the climb-back visible to the user and act on it; on an **invariant-erosion** finding, reopen the offending item and run the amendment escalation properly; on a **restructuring** verdict, transcribe the exact changes it returns verbatim — and if it calls for a whole new *area*, re-dispatch the Planner to scaffold it into `initial-agenda.md`, then mirror the new items into the living agenda. After it returns, **persist its pass** — append a `meta` row (its verdict + climb-back target or sanctioned restructuring) and set `last-facilitator: turn N` in the Position block.

### Dispatching the Tracer

The Facilitator reads the map; the Tracer *runs* it. Two resolved decisions can read as consistent side by side and still be jointly impossible the moment a concrete run threads through them — the class of defect no amount of re-reading catches. In this lane the Tracer traces **operational runs** through the resolved decisions, forcing state ownership at every step: where does the artifact physically live — before it's verified, before it's integrated, after a failure. Dispatch `grill-tracer` on two triggers:

- **Operational-cluster close:** when a cluster of interacting operational decisions closes — how work is isolated, how it's verified, how it's integrated, how it recovers — dispatch a scoped trace over just those decisions, while the user is still in that headspace.
- **Pre-completion gate:** a full trace across all areas, alongside the mandatory Facilitator pass (see *Completion*). This is the only place cross-area contradictions are visible end to end.

Give it only `lane: design`, the lane dir path, the posture, and the spec path. **Do not hand it a summary of the decisions or pick its scenarios for it** — it reads the state files itself and chooses its own scenarios, precisely because you are the one that drifts and would trace the comfortable path around the bug. It returns findings as exact lines in two buckets:

- **Blocking** — an execution contradiction (two decisions that collide on a concrete run). Apply it by transcribing the reopen lines verbatim (`<id> → [active]`); the colliding decisions go back into the interview. If the collision implicates a Locked entry, that's the escalation, not a reopen — the spec side of it is the user's call.
- **Surfaced (non-blocking)** — masked mechanisms, missing states/inputs, depth flags. Transcribe new concerns as `[unvisited]` nodes (re-dispatch the Planner for a whole new area); re-tag any self-blessed shallow resolve. These do **not** block completion — they become ordinary agenda items to resolve, defer, or record as an accepted risk (`[resolved: deferred]` with the reason) so a later trace won't re-raise it.

After the pass, persist it: append a `meta` row and, if it was the gate pass, note it toward READY. **Convergence is by categorization, not by zeroing findings:** a run completes when every concern is resolved, tagged `firm-up`/`deferred`, or surfaced as an open decision for the user. Only an *un-adjudicated execution contradiction* holds up completion, and the Tracer is capped at **two re-traces** per gate; leftover surfaced findings flow into the completion summary as known risks rather than looping.

## Completion

Before declaring the grill done, run the **mandatory pre-completion gate**: a Facilitator pass *and* a full Tracer pass. The Facilitator owns the **process** verdict — coverage, drift, posture-fit, invariant integrity; the Tracer owns the **substance** verdict — whether the resolved decisions collide when a concrete run threads through them. **READY requires both.** Don't conclude over the Facilitator's objection that material concern areas are untouched, or over an un-adjudicated execution contradiction the Tracer surfaced. Surfaced non-blocking findings need not be closed — but they must be visible in the summary below, not swept.

Then summarize for the user, reading position from `living-agenda.md` and the coverage diff against `initial-agenda.md`:

- what was **resolved**;
- which **assumptions changed** during the grilling;
- what **risks** surfaced;
- which **branches remain unexplored** (`unvisited` / `paused`);
- which **decisions still need a user call**;
- which resolved decisions are **`firm-up` or `deferred`** — the approximations and punts, plus any accepted risks the tracing surfaced, for /to-design to make precise or carry;
- which items are **`blocked-on-spec`** — the product questions that need a spec round-trip before their design can close;
- how the **givens** landed — each `consumed`, `superseded`, or explicitly `set-aside`; none left `[open]` (distinct from amendments: overturning a given never stales the spec);
- which Locked entries were **amended** — each spec id and its new decision. If any exist, say plainly: `spec.md` is stale — **re-run `/to-spec` before `/to-design`**.

Finally, **offer** — don't auto-run — the conversion: `/to-design` compiles this lane (plus the spec) into `grill-<slug>/design.md`. The grill produces understanding; converting it is the user's call.

## Templates

**`living-agenda.md`**

```markdown
# Design-Grill Working Agenda: <idea>

**Position**
- posture: <rung> — <optional note>
- active: <node id, or —>
- resume-target: <node id, or —>
- turns-since-node-change: 0
- last-facilitator: <turn N, or —>

**Locked (from spec)**
The entire spec.md is locked input; these are its sharpest edges. No agenda item may resolve against them.
- INV-001: <one-line non-negotiable, verbatim from spec.md>
- INV-002: <…> (amended) — <the superseding decision, in product terms>
- CON-001: <binding constraint, verbatim>

## B. Data & State — <one line>
- [unvisited] B1 <question to resolve>
- [resolved: mechanism] B2 <question> — <the concrete artifact/state/procedure it resolved to>
- [paused: blocked-on-spec OQ-002] B3 <question> — reason added: <standing rationale>
- [dropped: superseded by B1] B4 <question> — reason added: <standing rationale>
- [unvisited] B5 <question to resolve> — given G1 (decided): <one line>
```

**`conversation-path.md`**

```markdown
# Design-Grill Conversation Path: <idea>

| turn | active-node | move | summary | implication | drift | next-node |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | B1 | answer | event log is the source of truth | B2–B4 assume append-only store | none | B2 |
| 3 | C2 | amend | INV-003 amended: exports may be async | spec.md stale; re-run /to-spec | watch | C2 |
| 4 | C2 | meta | facilitator: drifting, 3 areas untouched | climb back to B4; close C2 | deep | B4 |
| 5 | B4 | given | filed G2: blue-green rollout (leaning) | K3 annotated; staying on B4 | none | B4 |
| 6 | D3 | meta | scout: Stripe webhooks not ordered (docs) | D3 can't assume ordering | none | D3 |
| 10 | F6 | meta | tracer: resume trace collides F1×K5 | reopen F1, K5; where partial state lives undecided | deep | F1 |
```

A `meta` row records a non-user event that changed your read of the map — Facilitator pass, Tracer pass, or Scout finding — exactly as in the spec grill: verdict in `summary`, the transcribed consequence in `implication`.

**`givens.md`**

```markdown
# Givens: <idea>

A-priori details the user brought. Entries are append-only; only the status token and the route mutate. `decided` = the user's settled call, held once confirmed; `leaning` = a preference, the default recommendation for its node.

- [open] G1 (decided, spec-lot) <one-line detail> → B5
- [consumed] G2 (leaning, turn 5) <one-line detail> → K3
- [set-aside] G3 (leaning, intake) <one-line detail> → —
```
