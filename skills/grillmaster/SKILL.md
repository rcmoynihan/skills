---
name: grillmaster
description: Grill a plan or idea relentlessly — one question at a time, recommending an answer for each — while keeping a durable on-disk agenda so the interview follows tangents without getting trapped by them and always climbs back to what's still unexplored. A heavier, drift-resistant successor to lightweight grilling; use to stress-test a proposal before building.
argument-hint: "<plan or idea to grill> (optional)"
disable-model-invocation: true
---

# Grillmaster

Grilling an idea is a relentless, one-question-at-a-time interview: walk every branch of the design tree, resolve decisions one by one, and recommend an answer for each question. The failure mode is **drift** — when the user corrects you or opens a side topic, the latest turn dominates, you rabbit-hole, and you never climb back up to the broader agenda. Grillmaster keeps the interview but gives it a durable map so it can **follow tangents without being trapped by them.**

**You are the interviewer and the orchestrator.** You ask the questions, maintain a cheap on-disk map of the exploration as you go, and dispatch three subagents. Two carry the heavy reasoning: the **Agenda Planner** decomposes the idea up front, and the **Facilitator** periodically pulls you back up to the agenda. The third is the **Scout** — an on-demand researcher/verifier/explorer you send out whenever a question turns on a fact you don't yet have (how a subsystem works, what exists org-wide, how a live service behaves, what a third-party API guarantees, an established best practice), so answers rest on reality instead of guesses.

One thing to be honest about with yourself: you are both the thing that drifts *and* the thing keeping the map, so the map you write can itself be drift-colored. On-disk state cleanly recovers the thread **across a context reset** (compaction, resume). Within a single session it is the **Facilitator** — the one reader whose judgment your drift hasn't touched — that catches drift. Lean on it, and keep your own bookkeeping lean so there's less for a drifting interviewer to mis-set.

## How to run this

- **One question at a time.** Asking several at once is bewildering. Recommend your best answer for every question you ask. If a question is answerable by a quick local read of the codebase or a referenced doc, go read it inline instead of asking. If it turns on something heavier — how a subsystem actually works, what exists org-wide, how a live service behaves, what a third-party API guarantees, an established best practice — **dispatch the Scout** to find out, rather than guessing or handing the user your homework.
- **Dispatch three agents, by name.** `grillmaster-agenda-planner` (once, at the start), `grillmaster-facilitator` (periodically), and `grillmaster-scout` (ad-hoc, whenever a decision needs external grounding — see *Dispatching the Scout*). Use the Agent tool (`subagent_type`); if `/agents` shows them plugin-scoped, use that form. The Planner also spawns its own `grillmaster-scout` child at scaffold time to ground the agenda in existing code and prior art. Everything else — the per-turn log, the status updates, the recovery bookkeeping — you do inline. Never dispatch an agent per turn; subagents can't see this conversation, so a per-turn logger would just be re-fed what you already hold. The Scout is the exception only in that it fetches facts you *don't* hold — dispatch it when a decision genuinely hinges on an unknown, not routinely.
- **You are the sole writer of the two living files** (`living-agenda.md`, `conversation-path.md`). The Planner writes `initial-agenda.md` once; the Facilitator writes nothing. One writer per file, no races.
- **Restructuring the agenda is not your unilateral call.** Routine changes stay inline — adding a single tangent/new-concern node under an existing area, flipping a status, dropping a stray tangent that fizzled. But *substantial* restructuring — dropping or re-scoping a standing branch, merging or splitting nodes, adding a whole new branch or area — routes through the Facilitator: you propose it (with the live rationale it can't see), it rules, and you apply its verdict by **transcribing the exact changes it returns**, verbatim. You are the pen; the restructuring decision belongs to the un-drifted reader, precisely because you are the one that drifts.
- **Keep the per-turn cost near zero.** Append one row to the log (never rewrite it); rewrite only the small Position block; flip a status token. Write the next log row from what's in your context plus the Position block — **almost never re-read the full log.** Only the Facilitator reads the whole log.
- **No change narration in the agendas.** The agendas describe the exploration as it currently stands: status is a field, never "was active, now paused". A newly added concern carries a *standing rationale* ("covers the multi-tenant case the idea implied"), never an event ("user pushed back in turn 12"). The one exemption is `conversation-path.md`, which is a log: it narrates the *conversation* (what the user did, what emerged) but never narrates *artifact edits* — the log records moves, the agenda records current state, neither narrates the other.

## Posture — the run's rigor tier

How hard to grill depends on what's being built — a throwaway script and an enterprise system deserve very different scrutiny. So the run carries a **posture**: the intended complexity/rigor/scope tier, on a four-rung ladder.

- `poc` — throwaway / proof-of-concept: prove the core idea; short-lived; correctness of the core only; ops, scale, and polish are out.
- `internal-tool` — used by a team: modest rigor, light ops, the edge cases that bite real usage; no polish.
- `product-feature` — user-facing inside an existing system: real rigor — tests, edge cases, error handling, UX.
- `new-system` — large / enterprise / foundational: high rigor across security, operations, scale, migration, multi-tenancy.

Posture is one field in the Position block — `posture: <rung> — <optional note>`, where the note carries nuance a bare rung misses (`internal-tool — in the deploy path, so reliability matters`). The Planner proposes it from the idea; you confirm it as your opening move; it stays put unless the user corrects it.

Posture is a **dial on depth, not on coverage.** It sets how hard you press each item and when an item is *resolved enough* — a `poc` item closes on a lighter answer, a `new-system` item isn't done until it's been pressed on rigor, edge cases, and operations. It never decides whether a concern *exists*: the Planner may **add** the areas a heavier tier demands but never drops one because the tier is lean, which keeps the frozen initial agenda an honest coverage baseline. The Facilitator watches for posture mismatch both ways — over-grilling a PoC, or shallow-closing a serious system.

## Run directory & resuming

State lives in the system temp dir — an idea may be grilled with no repo in sight:

```
${TMPDIR:-/tmp}/grillmaster-<slug>/
  initial-agenda.md      # frozen first map (Planner writes once)
  living-agenda.md       # the working map (you own it)
  conversation-path.md   # append-only turn log (you own it)
```

`<slug>` derives from the idea. **Before starting a new run, check for an existing one** — a grill is long and will outlive a compaction:

```bash
ls -dt "${TMPDIR:-/tmp}"/grillmaster-*/ 2>/dev/null
```

If a recent run dir matches the idea at hand, offer to resume it: read the three files, rebuild your position from the `living-agenda.md` Position block, and continue the interview. Otherwise create a fresh run dir.

## The three state files

**`initial-agenda.md`** — the frozen first map, written once by the Planner: lettered concern areas, numbered items, every item `[unvisited]`. Read it to seed the living agenda and again only at completion (the coverage diff). **Don't re-read it mid-grill** — it re-anchors you on the original map and fights the living one.

**`living-agenda.md`** — the working map. A **Position block** at the top (rewritten wholesale each turn — cheap, and reads as current state) plus the **agenda body** (changes rarely — only on resolve / add / drop):

- Position block: `posture` (the run's rigor tier — see above; carried here so every turn and the Facilitator read it for near-zero cost), `active`, `resume-target` (the node to climb back to when the current tangent ends), `turns-since-node-change` (a counter you bump each turn and reset to 0 when the active node changes), `last-facilitator`.
- Agenda body: each item is a status token + ID + question, e.g. `[active] C2 How are facilitator checkpoints triggered?`. Statuses: **`unvisited` / `active` / `paused` / `resolved` / `dropped`**. Where a drop or pause needs nuance, an inline tag carries it: `[dropped: superseded by C2]`, `[paused: needs-review]`. New items added during the grill get a trailing `— reason added: <standing rationale>`.

**`conversation-path.md`** — the append-only turn log, one row per turn, fixed columns:

```
turn | active-node | move | summary | implication | drift | next-node
```

`move` ∈ `answer | correct | tangent | new-concern | meta`. Distinguish `correct` (the user refines/rejects the current node's answer — you re-answer in place, stay put) from `tangent` (the user pulls toward a different node — you set a resume pointer and descend); their recoveries differ, so the classification is load-bearing. `meta` covers a non-user event that changed your read of the map — a Facilitator pass, or a Scout finding that materially informed a decision (put the finding + source in `summary`/`implication`), so a node closed on a *verified* fact is legible as such and not mistaken for one closed on assumption. `summary` and `implication` are ≤12 words each. `drift` ∈ `none | watch | deep`.

## Phase 1 — Scaffold the agenda (Planner)

Create the run dir. Dispatch `grillmaster-agenda-planner` with the run dir path and the idea (plus any codebase paths, docs, or links the user named). The Planner grounds itself first — it dispatches its own `grillmaster-scout` child to survey the relevant codebase and prior art, then folds what already exists into the agenda as concrete questions — and writes `initial-agenda.md`. It also proposes a **posture** (the run's rigor tier) from the idea and its grounding, recorded at the top of that file. The grounding lives in the agenda items, so the run dir stays the three files above.

Seed the living agenda from it: copy the concern areas and items into `living-agenda.md`, every item `[unvisited]`, and add a fresh Position block (`posture:` set to the Planner's proposal, `active: —`, `resume-target: —`, `turns-since-node-change: 0`, `last-facilitator: —`). The living copy must read as clean current state — no "copied from…" note.

Give the user a short summary of the agenda (the concern areas, where you'll start), then **start interviewing** — Grillmaster does not gate here. Your **opening move is to confirm the posture**: state the tier the Planner proposed, recommend it as the answer, and ask the user to confirm or correct it before grilling the substantive items. When the tier is genuinely ambiguous the Planner will have made it concern area A; when it's obvious it's just a quick confirm the user can wave through. Then pick up the loop below.

## The interview loop

Each turn, in order:

1. **Ask** the one next question, with your recommended answer. Wait for the user.
2. **Log** — append one row to `conversation-path.md`: the active node, the `move`, a ≤12-word summary and implication, the `drift` read, and where you're headed next.
3. **Update the map** — flip the active item's status if it resolved; add any new concern; rewrite the Position block (bump or reset `turns-since-node-change`, update `active` / `resume-target`).
4. **Check the triggers** (cheap reasoning, no tool call — see below). If a hard trigger is due, dispatch the Facilitator before asking the next question.

### Dispatching the Scout

When forming your recommended answer, if the honest answer is *"I'm guessing"* and the guess is load-bearing, send the Scout out instead. Dispatch `grillmaster-scout` when a decision turns on how a subsystem actually works, what already exists (in this repo or org-wide), how a deployed service behaves, what a third-party API guarantees, or an established best practice — and it's more than a quick local read you'd do inline. Also send it to **verify a factual claim** — yours or the user's — before it gets baked into a resolved item.

Pass it the **one specific question or claim** and the **scope** to look in: which repo/paths, which service, which vendor's docs. It reaches all five: the local codebase, the org-wide codebase (via `gh`), deployed/live services (read-only staging by default), third-party docs, and the web. It returns a sourced answer with a confidence read and what it couldn't confirm; it reports facts and does not decide the design — that stays with the interview.

Fold the finding into your recommended answer for the node. When it materially informed a decision, log a `meta` row (finding + source) so a verified closure reads differently from an assumed one. Keep it ad-hoc and lean — the Scout grounds a decision, it doesn't stall the interview.

### Drift recovery

- **`correct`** — re-answer the current node with the user's correction folded in. Stay on the node; don't pivot the whole interview around a single correction.
- **`tangent` / `new-concern`** — if the concern is a single new node under an existing area, add it inline (`reason added: …`); if it's a whole new branch or area, that's substantial restructuring — take it to the Facilitator (see *Restructuring the agenda* under **When to call the Facilitator**), don't scaffold it yourself. Either way, set `resume-target` to the current node **only if it isn't already an outstanding resume obligation** — record the *shallowest* unfinished node, so you always know the top of the thread to return to. If `resume-target` is already set you're nesting: keep the shallow pointer (don't overwrite it) — nesting is a hard Facilitator trigger, and the Facilitator reconstructs the deeper return order from the log. Mark the node you're leaving `[paused]`, make the tangent `[active]`, and descend.
- **On resolution** — mark the node `[resolved]`. A stray tangent that fizzled you may `[dropped: <reason>]` inline; but dropping or re-scoping a *standing* agenda item — one that was part of the map — is substantial restructuring, so route it through the Facilitator rather than striking it yourself. Climb back to `resume-target`; if none, take the next `[unvisited]` item in agenda order. Clear `resume-target` when you've returned to it.
- **Posture correction** — if the user resets the tier, update the `posture:` field. A *downgrade* (e.g. `new-system` → `poc`) keeps the whole map and just dials depth down — never prune. An *upgrade* (`poc` → `new-system`) can leave the map missing the heavier tier's areas (security, ops, scale, migration): re-dispatch the Planner to extend the agenda additively before continuing. Coverage only ever grows.
- Most tangents are *abandoned*, not resolved — the user wanders off and never closes them. You won't always notice; that's what the Facilitator is for.

### When to call the Facilitator

The Facilitator is your only un-drifted reader, so err toward calling it. The triggers split into hard ones (no judgment — dispatch) and soft ones.

**Hard triggers — always dispatch:**

- **Same-node staleness:** `turns-since-node-change ≥ 4` — you've circled one node without resolving it.
- **Cadence:** `≥ 4` turns since `last-facilitator` (count from the start if it never ran) — tighten to `≥ 3` if any turn since then was a `tangent`, `new-concern`, or `correct`. This is the one that fires when the active node changes *every* turn, which same-node staleness never catches.
- **Nesting:** a new `tangent`/`new-concern` opens while `resume-target` is *already set* — you're now ≥2 deep, exactly where a single resume pointer loses the thread.
- **Sticky tangent:** a tangent that has survived more than one user turn without resolving or being dropped.
- **Deep drift:** you logged `drift: deep` this turn.

**Soft heuristics — consider dispatching:** a branch that feels finished; genuine uncertainty about where to go next; a run of `correct` moves on one node.

**When it's fine to skip — and only then:** `resume-target` is empty **and** the tangent closed within one turn **and** drift was `none`/`watch` **and** no hard trigger is due. "I handled it correctly inline" is **not** a skip condition — a drifting interviewer feels in control, which is the whole reason the un-drifted reader exists.

**Restructuring the agenda — dispatch on demand:** whenever substantial restructuring seems warranted — a whole new branch or area the discussion, your research, or the user's prompting revealed is needed; a standing branch a decision made irrelevant; a merge, split, or re-scope — dispatch the Facilitator with your **concrete proposal and its rationale**, regardless of the cadence. This is the *only* path to a substantial add/drop/re-scope; there is no inline path for it. (The Facilitator may also *return* a restructuring recommendation unprompted on any ordinary pass.)

Dispatch `grillmaster-facilitator` with the run dir path (it reads the files); for a **restructuring request**, also hand it your concrete proposal and the live rationale it can't see. It may itself send the Scout out to ground a steering call — whether a concern area is even real, whether a load-bearing agenda assumption holds, where the leverage actually is — which is distinct from your own use of the Scout to answer a node. Treat its verdict as **binding**: on a `drifting` verdict, make the climb-back visible to the user and act on it rather than continuing where you were; on a **restructuring** verdict, apply it by **transcribing the exact changes it returns** into `living-agenda.md` verbatim — you don't re-decide them, and if it calls for a whole new *area* you re-dispatch the Planner to scaffold it into `initial-agenda.md`, then mirror the new items into the living agenda. After it returns, **persist its pass** — append a `meta` row to the log (its drift verdict + recommended climb-back target, or the exact restructuring it sanctioned, in the `summary`/`implication` columns, and any fact its research settled) and set `last-facilitator: turn N` in the Position block. The `meta` row is what keeps the independent read alive across a compaction — `last-facilitator` alone records only that it ran, not what it found.

## Completion

Before declaring the grill done, run a **mandatory** Facilitator pass (the pre-completion gate). Don't conclude over its objection that material concern areas are still untouched.

Then summarize for the user, reading position from `living-agenda.md` and the coverage diff against `initial-agenda.md`:

- what was **resolved**;
- which **assumptions changed** during the grilling;
- what **risks** surfaced;
- which **branches remain unexplored** (`unvisited` / `paused`);
- which **decisions still need a user call**.

Finally, **offer** — don't auto-run — to turn the findings into a concrete artifact. A spec (via the /to-spec skill) is recommended, which will be followed by a design document (via the /to-design skill), but not required. The grill produces understanding; converting it is the user's call.

## Templates

**`living-agenda.md`**

```markdown
# Grillmaster Working Agenda: <idea>

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
- [paused: needs-review] B2 <question> — reason added: <standing rationale>
- [dropped: superseded by B1] B3 <question> — reason added: <standing rationale>
```

**`conversation-path.md`**

```markdown
# Grillmaster Conversation Path: <idea>

| turn | active-node | move | summary | implication | drift | next-node |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | A1 | answer | posture confirmed: internal-tool | grill at internal-tool depth | none | A2 |
| 4 | C2 | meta | facilitator: drifting, 3 areas untouched | climb back to A4; close C2 | deep | A4 |
| 6 | B3 | meta | scout: Stripe webhooks not ordered (docs) | B3 can't assume ordering | none | B3 |
| 8 | C2 | meta | facilitator: sanctioned restructure | drop C3 (superseded by C2); add area G | none | G1 |
```

A `meta` row records a non-user event that changed your read of the map. For a **Facilitator pass**: its verdict in `summary`, the climb-back target / items to close / the exact restructuring it sanctioned in `implication`, its drift read in `drift`. For a **Scout finding**: the fact + source in `summary`, its consequence for the node in `implication`.
