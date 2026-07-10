---
name: spec-grill
description: Grill the product side of an idea relentlessly — one question at a time, recommending an answer for each — while keeping a durable on-disk agenda so the interview follows tangents without getting trapped by them. Exclusively non-technical — what it is, who it's for, what it must do; technical asides are parked for the design grill, never discussed. The first link in spec-grill → to-spec → design-grill → to-design, optionally seeded upstream by opp-grill → to-idea.
argument-hint: "<idea or plan to grill> (optional)"
disable-model-invocation: true
---

# Spec-Grill

Grilling an idea is a relentless, one-question-at-a-time interview: walk every branch of the product's decision tree, resolve decisions one by one, and recommend an answer for each question. This grill covers **the product side only** — the *what*: problem, actors, scope, journeys, behavior, promises. The technical *how* belongs to its sibling `/design-grill`, run after this grill's output is compiled by `/to-spec`. The failure mode is **drift** — when the user corrects you or opens a side topic, the latest turn dominates, you rabbit-hole, and you never climb back up to the broader agenda. Spec-grill keeps the interview but gives it a durable map so it can **follow tangents without being trapped by them.**

**You are the interviewer and the orchestrator.** You ask the questions, maintain a cheap on-disk map of the exploration as you go, and dispatch four subagents. Three carry the heavy reasoning: the **Planner** (`grill-planner`) decomposes the idea up front, the **Facilitator** (`grill-facilitator`) periodically pulls you back up to the agenda, and the **Tracer** (`grill-tracer`) runs concrete user journeys through your resolved decisions to catch the contradictions that surface only when an actor walks the product end to end. The fourth is the **Scout** (`grill-scout`) — an on-demand researcher/verifier you send out whenever a question turns on a fact you don't yet have (what users actually do, what the domain's prior art promises, what a dependency provides), so answers rest on reality instead of guesses.

One thing to be honest about with yourself: you are both the thing that drifts *and* the thing keeping the map, so the map you write can itself be drift-colored. On-disk state cleanly recovers the thread **across a context reset** (compaction, resume). Within a single session it is the two independent readers your drift hasn't touched that keep you honest, aimed at different targets: the **Facilitator** catches *process* drift — circling, rabbit-holing, branches left unpopped, technical bleed — by reading the map; the **Tracer** catches *product unsoundness* — decisions that read as consistent but collide on a concrete journey — by executing it. Lean on both, and keep your own bookkeeping lean so there's less for a drifting interviewer to mis-set.

## The lane boundary — product only, technical parks

This grill resolves product decisions and nothing else. The test, applied as a reflex to every question you're about to ask and every answer you're about to record, is the same one `/to-spec` and `/to-design` use to route content: **would this change if we re-implemented the same product a different way?** No → it's product — in scope here. Yes → it's technical — park it (below) and return to the product question.

| In scope (product) | Parked (technical) |
| --- | --- |
| Problem framing; who it's for (actors/personas) | Architecture, components, service boundaries |
| Scope: what's in, what's out | Storage, schema, technology, algorithm choices |
| User journeys and stories | Data/state mechanisms, caching, queues |
| Behavioral requirements; user-visible edge cases | Wire shapes, field types, error codes |
| The external-interface *promise* (what operations exist, what they mean) | The interface *realization* (schemas, idempotency, versioning mechanics) |
| NFR *targets* (the bound the user needs) | The *mechanism* that meets the target |
| Product tradeoffs and priorities | Deployment, ops, observability mechanics |
| Dependencies as *what is required* | Integration design (clients, pooling, failover) |
| Validation: how we'd know it works | Implementation sequencing |

Two guard rules:

- **Target vs. mechanism.** "Search must feel instant" is a product target — resolve it here. "Cache results in Redis" is a mechanism — park it. When an answer starts naming the machinery that delivers a promise, you've crossed the line.
- **No laundering.** A technical choice rephrased in product words is still out of lane — "the user expects their data to live in one place per region" is a data-residency *requirement* only if the user actually needs it; if it's a database topology preference wearing a product costume, park it.

## How to run this

- **One question at a time.** Asking several at once is bewildering. Recommend your best answer for every question you ask. If a question is answerable by a quick local read of the codebase or a referenced doc, go read it inline instead of asking. If it turns on something heavier — what the domain's prior art does, how users behave, what a dependency actually provides — **dispatch the Scout** to find out, rather than guessing or handing the user your homework.
- **Dispatch four agents, by name.** `grill-planner` (once, at the start), `grill-facilitator` (periodically), `grill-tracer` (when a cluster of product decisions closes, and at the pre-completion gate — see *Dispatching the Tracer*), and `grill-scout` (ad-hoc, whenever a decision needs external grounding — see *Dispatching the Scout*). Use the Agent tool (`subagent_type`); if `/agents` shows them plugin-scoped, use that form. **Every dispatch names the lane and the lane dir:** these are shared workers serving both grills, so each prompt carries `lane: spec` and the lane dir path (plus the posture, and the idea/docs where relevant) — subagents can't see this conversation. The Planner also spawns its own `grill-scout` child at scaffold time to ground the agenda in the domain and prior art. Everything else — the per-turn log, the parking lot, the givens ledger, the status updates, the recovery bookkeeping — you do inline. Never dispatch an agent per turn; a per-turn logger would just be re-fed what you already hold. The Scout and the Tracer fetch or test what you *can't* settle yourself — dispatch them on their triggers, not routinely.
- **You are the sole writer of the four living files** (`living-agenda.md`, `conversation-path.md`, `technical-parking-lot.md`, `givens.md`). The Planner writes `initial-agenda.md` once and only *reads* the givens ledger — it reports each given's route and you transcribe it; the Facilitator and the Tracer write nothing — they return exact lines you transcribe. One writer per file, no races.
- **Restructuring the agenda is not your unilateral call.** Routine changes stay inline — adding a single tangent/new-concern node under an existing area, flipping a status, dropping a stray tangent that fizzled. But *substantial* restructuring — dropping or re-scoping a standing branch, merging or splitting nodes, adding a whole new branch or area — routes through the Facilitator: you propose it (with the live rationale it can't see), it rules, and you apply its verdict by **transcribing the exact changes it returns**, verbatim. You are the pen; the restructuring decision belongs to the un-drifted reader, precisely because you are the one that drifts.
- **Keep the per-turn cost near zero.** Append one row to the log (never rewrite it); rewrite only the small Position block; flip a status token; append a parking-lot or givens line (plus its node annotation) when one lands; flip a given to `[consumed]` when its node resolves with it folded in. Write the next log row from what's in your context plus the Position block — **almost never re-read the full log.** Only the Facilitator reads the whole log.
- **No change narration in the agendas.** The agendas describe the exploration as it currently stands: status is a field, never "was active, now paused". A newly added concern carries a *standing rationale* ("covers the multi-tenant case the idea implied"), never an event ("user pushed back in turn 12"). The one exemption is `conversation-path.md`, which is a log: it narrates the *conversation* (what the user did, what emerged) but never narrates *artifact edits* — the log records moves, the agenda records current state, neither narrates the other.

## Posture — the run's rigor tier

How hard to grill depends on what's being built — a throwaway script and an enterprise system deserve very different scrutiny. So the run carries a **posture**: the intended complexity/rigor/scope tier, on a four-rung ladder.

- `poc` — throwaway / proof-of-concept: prove the core idea; short-lived; correctness of the core only; ops, scale, and polish are out.
- `internal-tool` — used by a team: modest rigor, light ops, the edge cases that bite real usage; no polish.
- `product-feature` — user-facing inside an existing system: real rigor — tests, edge cases, error handling, UX.
- `new-system` — large / enterprise / foundational: high rigor across security, operations, scale, migration, multi-tenancy.

Posture is one field in the Position block — `posture: <rung> — <optional note>`, where the note carries nuance a bare rung misses (`internal-tool — in the deploy path, so reliability matters`). The Planner proposes it from the idea; you confirm it as your opening move; it stays put unless the user corrects it. This confirmed posture travels the whole chain: `/to-spec` writes it into the spec's frontmatter, and `/design-grill` inherits it from there.

Posture is a **dial on depth, not on coverage.** It sets how hard you press each item and when an item is *resolved enough* — a `poc` item closes on a lighter answer, a `new-system` item isn't done until it's been pressed on the edge cases and promises the tier demands. It never decides whether a concern *exists*: the Planner may **add** the areas a heavier tier demands but never drops one because the tier is lean, which keeps the frozen initial agenda an honest coverage baseline. The Facilitator watches for posture mismatch both ways — over-grilling a PoC, or shallow-closing a serious system.

## Run directory & resuming

The whole chain — this grill, its compiled spec, the design grill, and the design doc — lives in **one run directory** in the system temp dir; an idea may be grilled with no repo in sight. The run dir is the run's identity: it is created once — by `/opp-grill` when the chain starts upstream, otherwise here — and every skill after it discovers it rather than re-deriving a slug.

```
${TMPDIR:-/tmp}/code-goblin-pro/grill-<slug>/
  idea/                        # the opportunity lane, when the chain started at /opp-grill
  idea.md                      # its compiled brief (written by /to-idea)
  spec/                        # this grill's lane
    initial-agenda.md          # frozen first map (Planner writes once)
    living-agenda.md           # the working map (you own it)
    conversation-path.md       # append-only turn log (you own it)
    technical-parking-lot.md   # deflected technical asides (you own it)
    givens.md                  # a-priori details the user brought (you own it)
  spec.md                      # written later by /to-spec
  design/                      # written later by /design-grill
  design.md                    # written later by /to-design
```

`<slug>` derives from the idea. **Before starting a new run, check for an existing one** — a grill is long and will outlive a compaction:

```bash
ls -dt "${TMPDIR:-/tmp}"/code-goblin-pro/grill-*/ 2>/dev/null
```

Read the matching run dir's contents to place it in the chain:

- `idea.md` exists but no `spec/living-agenda.md` — an upstream idea brief awaits its spec grill. Join this run dir — never mint a new slug — and consume the brief at intake (Phase 1). A brief whose frontmatter verdict is not `go` normally ended the chain there; confirm before grilling a spec on top of it.
- `idea/living-agenda.md` exists but no `idea.md` — an opp-grill in flight. Offer to finish upstream first — resume `/opp-grill` or compile `/to-idea` — rather than grilling the spec with the brief unwritten.
- `spec/living-agenda.md` exists but no `spec.md` — a spec grill in flight. Offer to resume: read the lane's files, rebuild your position from the `living-agenda.md` Position block, and continue the interview; `givens.md`'s `[open]` entries are standing details to re-hold (`decided` held as the user's call, `leaning` as the default recommendation for its node).
- `spec.md` (or `design/`, or `design.md`) exists — this run's spec lane is already compiled. Starting a new spec grill here is a **re-grill**: warn that `spec.md`, `design/`, and `design.md` go stale and confirm before proceeding.
- No run dir matches the idea — create a fresh `grill-<slug>/` with an empty `spec/`.

## The five state files

**`initial-agenda.md`** — the frozen first map, written once by the Planner: lettered concern areas, numbered items, every item `[unvisited]`. Read it to seed the living agenda and again only at completion (the coverage diff). **Don't re-read it mid-grill** — it re-anchors you on the original map and fights the living one.

**`living-agenda.md`** — the working map. A **Position block** at the top (rewritten wholesale each turn — cheap, and reads as current state) plus the **agenda body** (changes rarely — only on resolve / add / drop):

- Position block: `posture` (the run's rigor tier — see above; carried here so every turn and the Facilitator read it for near-zero cost), `active`, `resume-target` (the node to climb back to when the current tangent ends), `turns-since-node-change` (a counter you bump each turn and reset to 0 when the active node changes), `last-facilitator`.
- Agenda body: each item is a status token + ID + question, e.g. `[active] C2 What does a reviewer see when a check fails?`. Statuses: **`unvisited` / `active` / `paused` / `resolved` / `dropped`**. A `resolved` item **carries a depth tag** recording how firm its answer is: `[resolved: mechanism]` (names a concrete behavior/promise/procedure the user can observe), `[resolved: firm-up]` (an approximation fine to make precise later in /to-spec — e.g. "~2–3 retries from the user's view"), or `[resolved: deferred]` (explicitly punted, or an accepted risk the user chose to carry). Where a drop or pause needs nuance, an inline tag carries it the same way: `[dropped: superseded by C2]`, `[paused: needs-review]`. New items added during the grill get a trailing `— reason added: <standing rationale>`.

**`conversation-path.md`** — the append-only turn log, one row per turn, fixed columns:

```
turn | active-node | move | summary | implication | drift | next-node
```

`move` ∈ `answer | correct | tangent | new-concern | park | given | meta`. Distinguish `correct` (the user refines/rejects the current node's answer — you re-answer in place, stay put) from `tangent` (the user pulls toward a different node — you set a resume pointer and descend); their recoveries differ, so the classification is load-bearing. `park` is the lane's own move: the user's turn was *wholly* a technical pull — you appended it to the parking lot and stayed put, map untouched (`next-node` = the same node). `given` is `park`'s in-lane twin: the turn was *wholly* an a-priori product detail aimed at a node that isn't active — you appended a givens line, annotated the target node in the living agenda, and stayed put (a detail about the *active* node is ordinary material for the current answer; a technical detail parks instead). A given riding along with a substantive answer keeps the turn's real move, with the filing noted in `implication` (e.g. "filed G4 → E2"). `meta` covers a non-user event that changed your read of the map — a Facilitator pass, a **Tracer pass** (put its verdict + the contradiction or the nodes it opened in `summary`/`implication`), or a Scout finding that materially informed a decision (put the finding + source in `summary`/`implication`), so a node closed on a *verified* fact is legible as such and not mistaken for one closed on assumption. `summary` and `implication` are ≤12 words each. `drift` ∈ `none | watch | deep`.

**`technical-parking-lot.md`** — the append-only ledger of technical asides deflected out of lane, one line per item. Nothing in it binds this lane or the spec; `/to-spec` compiles it into the spec's non-binding Carried-Forward Technical Notes section, and the design grill's Planner seeds candidate agenda items from it. A line may carry a trailing `[decided|leaning]` tag — the user's stated conviction about a technical detail; the tag binds nothing here, but the design grill picks the line up as a given. Append and move on — never expand an entry into a discussion.

**`givens.md`** — the ledger of a-priori details the user brought, at intake, mid-grill, or carried in from an upstream idea brief: one line per given, `- [open] G1 (decided, intake) <one-line detail> → B3` — status, stable ID, conviction + origin (`intake`, `turn N`, or `brief`), the detail, and the node it routes to (`→ —` until routed). Statuses: **`open` / `consumed` / `superseded` / `set-aside`**. Conviction: `decided` — the user's settled call, held once confirmed; `leaning` — a preference, the default recommendation for its node. Entries are append-only; only the status token and the route mutate — current state lives in the fields, never in appended notes.

### Parking a technical pull

When the user raises technical content — or your own drafted recommendation starts answering a technical question — park it: append one line to `technical-parking-lot.md` (`- <the technical question or idea, one line> (turn N, near <node-id>)`, with a trailing `[decided|leaning]` when the user stated conviction; details extracted at intake use `(intake)`), acknowledge in at most one sentence ("parked for the design grill"), and re-ask the product question you were on. **Never discuss the parked item, never recommend an answer for it** — a recommendation is a design decision made in the wrong lane. If the user insists on resolving it now, name the boundary: this grill produces the spec; `/design-grill` exists precisely for that question and will hold everything decided here as its ground.

Log it: a turn that was wholly a technical pull is `move: park` (stay on the current node). A technical aside riding along with a substantive product answer keeps the turn's real move, with the park noted in `implication` (e.g. "parked: storage choice").

## Phase 1 — Scaffold the agenda (Planner)

Create the run dir, then run **intake**: create `givens.md` and `technical-parking-lot.md` with just their headers. When the run dir holds `idea.md`, consume the brief first: each Constraints & Non-Negotiables line and each Carried-Forward product line becomes a givens line (origin `brief`, conviction tag preserved); each `[technical]`-tagged Carried-Forward line is copied into `technical-parking-lot.md` verbatim-in-spirit, conviction tag intact; its Assumptions & Open Risks are held for the Planner as risky-assumption seed material. Then extract every a-priori detail embedded in the user's invocation — the specifics that ride along with the idea rather than constitute it. Product details become givens lines (`[open]`, origin `intake`, route `→ —`); technical details become conviction-tagged parking-lot lines. Tag `decided` only when the user stated it as settled ("must", "we're using X"); otherwise `leaning` — when unsure, `leaning`: under-claiming costs a normal grill, over-claiming skips scrutiny.

Dispatch `grill-planner` with `lane: spec`, the lane dir path (`grill-<slug>/spec/`), the idea (plus any codebase paths, docs, or links the user named), and the givens path when any were filed. When `idea.md` exists it is the idea's primary statement: pass its path as the leading input doc, hand over its Assumptions & Open Risks as risky-assumption seed material, and name its frontmatter `posture_proposal` as the seed for the Planner's posture proposal. The Planner grounds itself first — it dispatches its own `grill-scout` child to survey the domain, its users, and prior art, then folds what already exists into the agenda as concrete product questions — and writes `initial-agenda.md`, threading each given into the item it informs. It also proposes a **posture** (the run's rigor tier) from the idea and its grounding, recorded at the top of that file. The grounding lives in the agenda items, so the lane dir stays the five files above. When the Planner returns, transcribe the routes it reports into the ledger's route fields.

Seed the living agenda from it: copy the concern areas and items into `living-agenda.md` (given annotations ride along), every item `[unvisited]`, and add a fresh Position block (`posture:` set to the Planner's proposal, `active: —`, `resume-target: —`, `turns-since-node-change: 0`, `last-facilitator: —`). The living copy must read as clean current state — no "copied from…" note.

Give the user a short summary of the agenda (the concern areas, where you'll start), list each intake given with its ID, conviction, and route (`G1 (decided) → B5: per-user export toggle`) — acknowledgment, not resolution: every node stays `[unvisited]`, and each `decided` given is confirmed when its node goes active, one per turn — and extend the standing invite: anything the user already knows they want, said now or any time, gets filed where it belongs. **Surface the prior art before diving in:** if the Planner's grounding turned up existing solutions that plausibly cover the need, name them to the user in a sentence or two — an honest early look, so a close match isn't discovered only after the grill. If one looks close enough, the user may decide to adopt it and stop rather than grill a spec for something to build; otherwise proceed, and `/prior-art-check` runs the rigorous needs-level match once `/to-spec` has compiled the spec. When `idea.md` is present, prior art was already surveyed upstream (opp-grill) and again by the Planner — restate that finding rather than commissioning a fresh survey. Then **start interviewing** — spec-grill does not gate here. Your **opening move is to confirm the posture**: state the tier the Planner proposed, recommend it as the answer, and ask the user to confirm or correct it before grilling the substantive items. When the tier is genuinely ambiguous the Planner will have made it concern area A; when it's obvious it's just a quick confirm the user can wave through. Then pick up the loop below.

## The interview loop

Each turn, in order:

1. **Ask** the one next question, with your recommended answer. Wait for the user.
2. **Sort the reply against the lane boundary** — technical content gets parked (above) before anything else happens.
3. **Log** — append one row to `conversation-path.md`: the active node, the `move`, a ≤12-word summary and implication, the `drift` read, and where you're headed next.
4. **Update the map** — flip the active item's status if it resolved (**at most one standing item per turn** — closing several at once is how a shallow answer slips through unexamined; if you're tempted to bulk-close, you're describing, not resolving), and flip its given to `[consumed]` when the resolution folded one in; add any new concern; rewrite the Position block (bump or reset `turns-since-node-change`, update `active` / `resume-target`).
5. **Check the triggers** (cheap reasoning, no tool call — see below). If a hard trigger is due, dispatch the Facilitator before asking the next question.

### Dispatching the Scout

When forming your recommended answer, if the honest answer is *"I'm guessing"* and the guess is load-bearing, send the Scout out instead. Dispatch `grill-scout` (with `lane: spec` and the lane dir) when a product decision turns on what users actually do, what the domain's established products promise, what a dependency provides at the *what* level, or a factual claim — yours or the user's — that should be verified before it gets baked into a resolved item. It's worth a dispatch when it's more than a quick local read you'd do inline.

Pass it the **one specific question or claim** and the **scope** to look in: which domain, which product, which vendor's docs. It reaches the local codebase, the org-wide codebase (via `gh`), deployed/live services (read-only staging by default), third-party docs, and the web. It returns a sourced answer with a confidence read and what it couldn't confirm; it reports facts and does not decide the product — that stays with the interview.

Fold the finding into your recommended answer for the node. When it materially informed a decision, log a `meta` row (finding + source) so a verified closure reads differently from an assumed one. Keep it ad-hoc and lean — the Scout grounds a decision, it doesn't stall the interview.

### Drift recovery

- **`correct`** — re-answer the current node with the user's correction folded in. Stay on the node; don't pivot the whole interview around a single correction.
- **`tangent` / `new-concern`** — if the concern is a single new node under an existing area, add it inline (`reason added: …`); if it's a whole new branch or area, that's substantial restructuring — take it to the Facilitator (see *Restructuring the agenda* under **When to call the Facilitator**), don't scaffold it yourself. Either way, set `resume-target` to the current node **only if it isn't already an outstanding resume obligation** — record the *shallowest* unfinished node, so you always know the top of the thread to return to. If `resume-target` is already set you're nesting: keep the shallow pointer (don't overwrite it) — nesting is a hard Facilitator trigger, and the Facilitator reconstructs the deeper return order from the log. Mark the node you're leaving `[paused]`, make the tangent `[active]`, and descend.
- **`park`** — not a tangent: the map doesn't move. Append the parking-lot line, stay on the node, re-ask the product question. If the same technical pull recurs across turns, that's a soft Facilitator signal — the user may be telling you a product concern is hiding under the technical one; let the un-drifted reader make that call.
- **`given`** — not a tangent either: append the givens line, annotate the target node in the living agenda, stay on the node, re-ask the question you were on. A given with no matching node gets a single new node inline under the right area (`reason added: …`); with no home area, file it `→ —` and take the area to the Facilitator via the restructuring path. When a given-annotated node goes active: a `leaning` given is your recommended answer, grilled at the posture's normal depth; a `decided` given is confirm-once-and-hold — restate it, voice disagreement at most once, and close on the user's confirm, still naming what it bottoms out in (a label-shaped given — "decided: handle errors gracefully" — holds the *choice* while you grill the mechanics beneath it). Once held, don't re-argue it; only a collision — a Tracer or Facilitator finding, or a later decision that contradicts it — reopens it, and the user adjudicates: keep the given and revise the colliding decision, or overturn it (a `correct` move; flip the ledger line to `[superseded]`).
- **On resolution** — before you mark a node `[resolved]`, name what its answer bottoms out in: the observable behavior, the promise a user can rely on, or the procedure that decides it. Tag the close `[resolved: mechanism]`, `[resolved: firm-up]`, or `[resolved: deferred]` (see the status list). An answer that only *renames the hard part* with a concrete-sounding phrase — "handle it gracefully", "make onboarding smooth", "support Y" — is not a resolution; keep it `[active]`. A load-bearing decision (one that appears in an invariant/requirement or constrains other decisions) must reach `mechanism` to close; only genuine downstream precision may rest at `firm-up`. A stray tangent that fizzled you may `[dropped: <reason>]` inline; but dropping or re-scoping a *standing* agenda item — one that was part of the map — is substantial restructuring, so route it through the Facilitator rather than striking it yourself. Climb back to `resume-target`; if none, take the next `[unvisited]` item in agenda order. Clear `resume-target` when you've returned to it.
- **Posture correction** — if the user resets the tier, update the `posture:` field. A *downgrade* (e.g. `new-system` → `poc`) keeps the whole map and just dials depth down — never prune. An *upgrade* (`poc` → `new-system`) can leave the map missing the heavier tier's areas: re-dispatch the Planner to extend the agenda additively before continuing. Coverage only ever grows.
- Most tangents are *abandoned*, not resolved — the user wanders off and never closes them. You won't always notice; that's what the Facilitator is for.

### When to call the Facilitator

The Facilitator is your only un-drifted reader, so err toward calling it. The triggers split into hard ones (no judgment — dispatch) and soft ones.

**Hard triggers — always dispatch:**

- **Same-node staleness:** `turns-since-node-change ≥ 4` — you've circled one node without resolving it.
- **Cadence:** `≥ 4` turns since `last-facilitator` (count from the start if it never ran) — tighten to `≥ 3` if any turn since then was a `tangent`, `new-concern`, or `correct`. This is the one that fires when the active node changes *every* turn, which same-node staleness never catches.
- **Nesting:** a new `tangent`/`new-concern` opens while `resume-target` is *already set* — you're now ≥2 deep, exactly where a single resume pointer loses the thread.
- **Sticky tangent:** a tangent that has survived more than one user turn without resolving or being dropped.
- **Deep drift:** you logged `drift: deep` this turn.

**Soft heuristics — consider dispatching:** a branch that feels finished; genuine uncertainty about where to go next; a run of `correct` moves on one node; the same technical pull parked more than twice; a given filed with no home node.

**When it's fine to skip — and only then:** `resume-target` is empty **and** the tangent closed within one turn **and** drift was `none`/`watch` **and** no hard trigger is due. "I handled it correctly inline" is **not** a skip condition — a drifting interviewer feels in control, which is the whole reason the un-drifted reader exists.

**Restructuring the agenda — dispatch on demand:** whenever substantial restructuring seems warranted — a whole new branch or area the discussion, your research, or the user's prompting revealed is needed; a standing branch a decision made irrelevant; a merge, split, or re-scope — dispatch the Facilitator with your **concrete proposal and its rationale**, regardless of the cadence. This is the *only* path to a substantial add/drop/re-scope; there is no inline path for it. (The Facilitator may also *return* a restructuring recommendation unprompted on any ordinary pass.)

Dispatch `grill-facilitator` with `lane: spec` and the lane dir path (it reads the files); for a **restructuring request**, also hand it your concrete proposal and the live rationale it can't see. It may itself send the Scout out to ground a steering call — whether a concern area is even real, whether a load-bearing agenda assumption holds, where the leverage actually is — which is distinct from your own use of the Scout to answer a node. Treat its verdict as **binding**: on a `drifting` verdict, make the climb-back visible to the user and act on it rather than continuing where you were; on a **lane-bleed** finding, transcribe the parking-lot lines it returns and re-open the laundered items it names; on a **restructuring** verdict, apply it by **transcribing the exact changes it returns** into `living-agenda.md` verbatim — you don't re-decide them, and if it calls for a whole new *area* you re-dispatch the Planner to scaffold it into `initial-agenda.md`, then mirror the new items into the living agenda. After it returns, **persist its pass** — append a `meta` row to the log (its drift verdict + recommended climb-back target, or the exact restructuring it sanctioned, in the `summary`/`implication` columns, and any fact its research settled) and set `last-facilitator: turn N` in the Position block. The `meta` row is what keeps the independent read alive across a compaction — `last-facilitator` alone records only that it ran, not what it found.

### Dispatching the Tracer

The Facilitator reads the map; the Tracer *runs* it. Two resolved product decisions can read as consistent side by side and still be jointly impossible the moment a concrete actor walks through them — the class of defect no amount of re-reading catches. In this lane the Tracer traces **user/actor journeys** through the resolved decisions: who does what, and what the product has promised they observe at every step. Dispatch `grill-tracer` on two triggers:

- **Decision-cluster close:** when a cluster of interacting product decisions closes — an actor's permissions, a flow's behavior, its error semantics, what neighboring actors see — dispatch a scoped trace over just those decisions, while the user is still in that headspace. Catching a collision here is cheap; catching it many turns later means unwinding everything built on top.
- **Pre-completion gate:** a full trace across all areas, alongside the mandatory Facilitator pass (see *Completion*). This is the only place cross-area contradictions — decisions that live in different areas — are visible end to end.

Give it only `lane: spec`, the lane dir path, and the posture (and the source proposal if there is one). **Do not hand it a summary of the decisions or pick its scenarios for it** — it reads the state files itself and chooses its own journeys, precisely because you are the one that drifts and would trace the comfortable path around the bug. It returns findings as exact lines in two buckets:

- **Blocking** — an execution contradiction (two decisions that collide on a concrete journey). Apply it by transcribing the reopen lines verbatim (`<id> → [active]`), exactly as you transcribe a Facilitator restructuring; the colliding decisions go back into the interview.
- **Surfaced (non-blocking)** — masked promises, missing journeys/actors, depth flags, and any technical gaps it hit (those come back as *suggested parking-lot lines*, never agenda nodes). Transcribe new concerns as `[unvisited]` nodes (re-dispatch the Planner for a whole new area); append its parking-lot suggestions; re-tag any self-blessed shallow resolve. These do **not** block completion — they become ordinary agenda items to resolve, defer, or, if the user judges the risk immaterial at this posture, **record as an accepted risk** (`[resolved: deferred]` with the reason) so a later trace won't re-raise it.

After the pass, persist it: append a `meta` row (the Tracer's verdict + the contradiction or the nodes it opened) and, if it was the gate pass, note it toward READY. **Convergence is by categorization, not by zeroing findings:** a run completes when every concern is resolved, tagged `firm-up`/`deferred`, or surfaced as an open decision for the user — not when an adversary runs out of things to say. Only an *un-adjudicated execution contradiction* holds up completion, and the Tracer is capped at **two re-traces** per gate; leftover surfaced findings flow into the completion summary as known risks rather than looping.

## Completion

Before declaring the grill done, run the **mandatory pre-completion gate**: a Facilitator pass *and* a full Tracer pass. The Facilitator owns the **process** verdict — coverage, drift, posture-fit, lane discipline; the Tracer owns the **substance** verdict — whether the resolved decisions collide when a concrete journey threads through them. **READY requires both:** all agenda items resolved and read-consistent (Facilitator), *and* the traces collision-free with no `[resolved: mechanism]` item failing the depth check (Tracer). Don't conclude over the Facilitator's objection that material concern areas are untouched, or over an un-adjudicated execution contradiction the Tracer surfaced. Surfaced non-blocking findings need not be closed — but they must be visible in the summary below, not swept.

Then summarize for the user, reading position from `living-agenda.md` and the coverage diff against `initial-agenda.md`:

- what was **resolved**;
- which **assumptions changed** during the grilling;
- what **risks** surfaced;
- which **branches remain unexplored** (`unvisited` / `paused`);
- which **decisions still need a user call**;
- which resolved decisions are **`firm-up` or `deferred`** — the approximations and punts, plus any accepted risks the tracing surfaced, for /to-spec to make precise or carry;
- how the **givens** landed — each `consumed` into a resolution, `superseded`, or explicitly `set-aside`; none left `[open]`;
- what was **parked** — the count of technical asides in `technical-parking-lot.md` and its path, the backlog heading into the design grill (conviction-tagged lines travel onward as givens).

Finally, **offer** — don't auto-run — the rest of the chain: `/to-spec` compiles this lane into `grill-<slug>/spec.md`; `/prior-art-check` then checks whether an existing solution already satisfies that spec before design cost is sunk; `/design-grill` grills the technical side against the spec; `/to-design` compiles the design doc. The grill produces understanding; converting it is the user's call.

## Templates

**`living-agenda.md`**

```markdown
# Spec-Grill Working Agenda: <idea>

**Position**
- posture: <rung> — <optional note>
- active: <node id, or —>
- resume-target: <node id, or —>
- turns-since-node-change: 0
- last-facilitator: <turn N, or —>

## A. Posture & Scope — the rigor tier and the in/out-of-scope boundary
- [active] A1 What tier is this — poc / internal-tool / product-feature / new-system?
- [unvisited] A2 <what's in scope, and what's explicitly out>

## B. <concern area> — <one line>
- [unvisited] B1 <question to resolve>
- [resolved: mechanism] B2 <question> — <the observable behavior/promise/procedure it resolved to>
- [paused: needs-review] B3 <question> — reason added: <standing rationale>
- [dropped: superseded by B1] B4 <question> — reason added: <standing rationale>
- [unvisited] B5 <question to resolve> — given G1 (decided): <one line>
```

**`conversation-path.md`**

```markdown
# Spec-Grill Conversation Path: <idea>

| turn | active-node | move | summary | implication | drift | next-node |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | A1 | answer | posture confirmed: internal-tool | grill at internal-tool depth | none | A2 |
| 3 | B2 | park | parked: user proposed Postgres schema | product question re-asked | none | B2 |
| 4 | C2 | meta | facilitator: drifting, 3 areas untouched | climb back to A4; close C2 | deep | A4 |
| 6 | B3 | meta | scout: competitors all auto-save drafts (web) | B3 default should match | none | B3 |
| 7 | B3 | given | filed G2: autosave cadence preference (leaning) | E2 annotated; staying on B3 | none | B3 |
| 8 | C2 | meta | facilitator: sanctioned restructure | drop C3 (superseded by C2); add area G | none | G1 |
| 10 | F6 | meta | tracer: viewer journey collides F1×K5 | reopen F1, K5; who sees drafts is undecided | deep | F1 |
```

A `meta` row records a non-user event that changed your read of the map. For a **Facilitator pass**: its verdict in `summary`, the climb-back target / items to close / the exact restructuring it sanctioned in `implication`, its drift read in `drift`. For a **Tracer pass**: its verdict in `summary` (collision-free, or the colliding ID pair), the reopened/added nodes in `implication`, and whether a contradiction blocks in `drift`. For a **Scout finding**: the fact + source in `summary`, its consequence for the node in `implication`.

**`technical-parking-lot.md`**

```markdown
# Technical Parking Lot: <idea>

Append-only. One line per item. Nothing here binds this lane or the spec — these seed the design grill's agenda; a conviction tag travels onward as a given.

- <one-line technical question or idea> (turn N, near <node-id>)
- <one-line technical detail the user has settled> (intake) [decided]
```

**`givens.md`**

```markdown
# Givens: <idea>

A-priori details the user brought. Entries are append-only; only the status token and the route mutate. `decided` = the user's settled call, held once confirmed; `leaning` = a preference, the default recommendation for its node.

- [open] G1 (decided, intake) <one-line detail> → B5
- [consumed] G2 (leaning, turn 7) <one-line detail> → E2
- [set-aside] G3 (leaning, intake) <one-line detail> → —
```
