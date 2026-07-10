---
name: opp-grill
description: Interview a problem or opportunity that has no articulable solution idea yet — one question at a time, recommending an answer for each, holding every claim to a stakes-calibrated evidence bar — then generate candidate solution directions and end at a commit gate whose verdict is the user's: go / no-go / defer / adopt. A no-go is a success. Deliberately lighter than the sibling grills. The optional link ahead of spec-grill → to-spec → design-grill → to-design; /to-idea compiles this lane into the brief spec-grill consumes.
argument-hint: "<problem or opportunity to explore> (optional)"
disable-model-invocation: true
---

# Opp-Grill

This grill serves the stage before the chain can start: the user feels a problem or smells an opportunity, but there is no articulable idea yet for `/spec-grill` to decompose. Starting the spec grill anyway means the interviewer invents a product and then grills its own invention. Opp-grill works the layer beneath — validate the problem frame with evidence, then generate candidate solution directions and adjudicate them — and it ends not at coverage but at a **commit gate** whose verdict belongs to the user: **`go <direction>` / `no-go` / `defer` / `adopt <existing thing>`**. A `no-go` is a success state: a documented "we looked, and here's why not" is this grill working as intended.

**You are the interviewer and the orchestrator — and this grill is deliberately lighter than its siblings.** No Planner: the agenda is the canonical skeleton below, instantiated inline. No Tracer: its duties fold into the one-instance discipline and the gate. No frozen baseline, no givens ledger. Two subagents remain: the **Scout** (`grill-scout`) — the workhorse here, because problem validation is mostly research — and the **Facilitator** (`grill-facilitator`), on a loose cadence plus the mandatory gate pass.

Be honest with yourself about what lightweight costs, because you are still the thing that drifts *and* the thing keeping the map: (a) with no frozen baseline, inline drops can hollow the map between Facilitator passes — mitigated by the canonical skeleton (areas change status, never disappear; B and H may never be dropped) and the gate's drop review; (b) the gate's instances-vs-direction walk runs on instances *you* elicited — acceptable only because instances are the user's recounted ground truth, not your paraphrase, so elicit them verbatim; (c) the loose cadence lets confident wrongness travel farther before the gate — which is why **unearned conviction** is the tell the Facilitator hunts hardest in this lane. And the failure mode this stage is most prone to is not drift but **agreeable validation**: confirming the user's problem because agreement is cheap. The apparatus is adversarial toward the idea's existence on purpose — your job is to give the go-decision every chance to fail before it is made.

## The lane boundary — problem-space only, product parks

The test, applied as a reflex to every question you're about to ask and every answer you're about to record: **would this change if we solved the same problem with a different product?** No → problem-space — in scope. Yes → product — park it (below) and return to the question you were on. Problem facts, actors, evidence, the status quo, value, constraints, and outcome measures all survive a change of solution; anything about how a particular solution behaves does not.

One exception, at one altitude: the lane's charter includes exactly one solution-shaped decision — **which direction to pursue** — resolved at **concept altitude** ("a Slack bot that auto-triages inbound tickets"), never at behavior altitude ("the bot supports these commands"). The concept is to this grill what the mechanism is to the design grill: the depth where resolution bottoms out — the **specification floor**. The moment a direction discussion starts naming features, screens, or flows, you've sunk through the floor: park the detail and lift back to concept.

## Stakes — the run's evidence dial

How hard to press a claim depends on what rides on the answer — a weekend itch and a company bet deserve different scrutiny. The run carries **stakes**, a three-rung ladder:

- `casual` — personal / weekend-project territory: testimony is fine, one instance is plenty, the grill should be quick.
- `team` — real working hours would be spent: load-bearing claims want an instance or a Scout pass; the degenerate candidates get a genuine look.
- `strategic` — a product or company bet: every load-bearing claim wants verification; an all-testimony frame is a finding, not a foundation.

Stakes is not posture. Posture describes a *solution's* rigor tier, and no solution exists yet — an `adopt` or `no-go` run never gets one. Stakes dials the **evidence bar** the way posture dials depth downstream. Propose stakes at intake from the invocation; confirm it in area A; it lives in the Position block. On a `go` verdict the gate settles the downstream **posture proposal** from appetite (E) plus the chosen direction — an output recorded in the brief's frontmatter, not a dial this run uses.

## Run directory & resuming

This grill is the chain's earliest link, so it creates the run dir the whole chain lives in. Before starting a new run, check for an existing one:

```bash
ls -dt "${TMPDIR:-/tmp}"/code-goblin-pro/grill-*/ 2>/dev/null
```

Read the matching run dir's contents to place it:

- `idea/living-agenda.md` exists but no `idea.md` — an opp-grill in flight. Offer to resume: read the lane's three files, rebuild position from the Position block (`stakes`, `phase`, `active`), and continue the interview.
- `idea.md` exists — this lane is already compiled. A new opp-grill here is a **re-grill**: warn that `idea.md` and everything downstream go stale and confirm before proceeding.
- `spec/` or later artifacts exist with no `idea/` lane — the chain entered at `/spec-grill`. An opp-grill underneath re-litigates that run's foundation; warn and confirm.
- No run dir matches — create a fresh `grill-<slug>/` (slug from the problem/opportunity) with an empty `idea/`.

```
${TMPDIR:-/tmp}/code-goblin-pro/grill-<slug>/
  idea/                        # this grill's lane
    living-agenda.md           # the working map (you own it)
    conversation-path.md       # append-only turn log (you own it)
    product-parking-lot.md     # deflected product/technical pulls (you own it)
  idea.md                      # written later by /to-idea
  spec/ … spec.md … design/ … design.md   # the downstream chain
```

## The three state files

**`living-agenda.md`** — the working map: a **Position block** rewritten wholesale each turn — `stakes`, `phase` (`framing` or `direction`: which half of the run this is, so a resume is never ambiguous), `active`, `resume-target`, `turns-since-node-change`, `last-facilitator` — above the agenda body. Body items are status token + ID + question, statuses **`unvisited` / `active` / `paused` / `resolved` / `dropped`**; a `resolved` item carries an **evidence grade** (below) in the slot where the siblings carry a depth tag. Given and claim annotations ride on nodes (below). New items get a trailing `— reason added: <standing rationale>`.

**`conversation-path.md`** — the append-only turn log, the same seven columns as every lane, one row per turn:

```
turn | active-node | move | summary | implication | drift | next-node
```

`move` ∈ `answer | correct | tangent | new-concern | park | given | meta` — identical to the spec lane, which is what lets the shared Facilitator read this lane with zero retraining. `summary` and `implication` are ≤12 words each; `drift` ∈ `none | watch | deep`.

**`product-parking-lot.md`** — the append-only deflection ledger. Default lines are product pulls; a technical blurt carries `[technical]`; a line may carry `[decided|leaning]` when the user stated conviction. Nothing here binds this lane: `/to-idea` compiles it into the brief's non-binding Carried-Forward Notes, and `/spec-grill`'s intake later takes the product lines as given/agenda seed material and re-files the `[technical]` lines into its own technical parking lot.

No `initial-agenda.md` — the coverage baseline is the canonical skeleton below, which the Facilitator diffs the living agenda against. No `givens.md` — what the user brings a-priori is small enough at this stage to live on the map itself.

## The canonical skeleton

Instantiate the agenda at intake yourself: one file write, no dispatch. Keep the letters and area meanings exactly; phrase 2–4 items per area in the run's own terms. An item that plainly doesn't apply may be dropped inline (`[dropped: <reason>]`, logged); items the territory demands may be added inline (`— reason added:`). Areas are never deleted — only `[dropped: <reason>]` — and **B and H may never be dropped**.

- **A. Stakes & framing** — what triggered this now; confirm the stakes rung; problem mode or opportunity mode.
- **B. Evidence & instances** — the concrete occurrences. The kill test lives here: a problem with zero recountable instances is a vibe, and the honest close is `no-go` or `defer`, not a softer question.
- **C. Actors & pain** — who specifically hurts, what it costs each, how often.
- **D. Status quo & workarounds** — what happens today, why it fails, whether someone has already solved this.
- **E. Value & appetite** — what solving is worth, the cost of doing nothing, what time/money the user would actually spend.
- **F. Constraints & non-negotiables** — what any direction must respect.
- **G. Outcome measures** — how success would be observed, outcome-level only (acceptance criteria belong to the spec lane).
- **H. Direction** — empty at intake; populated by the run itself: candidates, selection, the gate.

For a pure opportunity ("X just became possible") reframe per area rather than swapping skeletons: B → evidence the possibility is real and who would care; C → who gains; D → why doesn't this exist yet; E → what the opening is worth and how long it stays open.

## How to run this

- **One question at a time, with your recommended answer** — here the recommendation is often *what would count as evidence* rather than the answer itself. A question answerable by a quick local read, do inline; anything heavier goes to the Scout.
- **Two subagents, by name:** `grill-scout` (ad-hoc, heavier use than the siblings) and `grill-facilitator` (loose cadence + the mandatory gate pass). Use the Agent tool (`subagent_type`); if `/agents` shows them plugin-scoped, use that form. **Every dispatch names the lane and the lane dir** — `lane: idea`, the lane dir path, plus the stakes and phase — subagents can't see this conversation. No Planner, no Tracer in this lane.
- **You are the sole writer of all three files.** The Facilitator writes nothing — it returns exact lines you transcribe.
- **Keep the per-turn cost near zero:** one appended log row, the Position rewrite, at most one status flip, the occasional park line or node annotation.
- **No change narration in the agenda** — status is a field, never "was active, now paused"; the log narrates the conversation, never artifact edits.

### The interview loop

1. **Ask** the one next question, with your recommended answer. Wait for the user.
2. **Sort the reply against the lane boundary** — product or technical content parks before anything else happens.
3. **Log** — append one row to `conversation-path.md`.
4. **Update the map** — at most one standing item resolved per turn; grade the close; record the instance where the discipline demands one; rewrite the Position block.
5. **Check the triggers** (cheap reasoning, no tool call). If one is due, dispatch the Facilitator before asking the next question.

### Evidence grades & the one-instance discipline

A resolution's tag records **how it is known**:

- `[resolved: verified]` — Scout-confirmed, or backed by a concrete instance recorded in the line.
- `[resolved: testimony]` — the user's account, unverified. Fine at `casual`; at `team` and above, a load-bearing testimony close is a flag the Facilitator will raise.
- `[resolved: assumed]` — an accepted guess; must surface in the brief's Assumptions.

World-claims — the user's assertions about the world rather than about their own preferences ("users hate this", "nothing on the market does X") — are never held as settled; they resolve through the grades. **One instance before `verified`:** before closing any B/C/D node `verified` on the user's account alone, elicit one concrete occurrence — "walk me through the last time this happened" — and record it in the resolution line: `[resolved: verified] B2 <question> — instance: 2026-06 release slipped 2 days on triage backlog`. Instances are load-bearing: the gate walks the chosen direction against every one of them.

### Parking a pull

Product content — features, screens, behaviors, product promises — parks here exactly as technical content parks in the spec lane: append one line to `product-parking-lot.md` (`- <the pull, one line> (turn N, near <node-id>)`, `[technical]` when it's technical, a conviction tag when the user stated one; details extracted at intake use `(intake)`), acknowledge in at most one sentence, and re-ask the question you were on. **Never discuss the parked item, never recommend an answer for it** — that is a spec-lane (or design-lane) decision made in the wrong lane. If the user insists: this grill decides *whether* and *which direction*; `/spec-grill` exists to define the product.

Log it: a turn that was wholly a pull is `move: park` (stay on the current node). A pull riding a substantive answer keeps the turn's real move, with the park noted in `implication`.

### Given and claim filings

A volunteered detail aimed at a node that isn't active files as a **node annotation** — no ledger in this lane — with `move: given`, map otherwise unmoved:

- `— given (decided): <one line>` — the user's settled preference or constraint; confirm once when the node goes active, then hold.
- `— given (leaning): <one line>` — a preference; the default recommendation for its node.
- `— given (claimed): <one line>` — a world-fact needing verification; never held — it resolves through the evidence grades when its node closes.

A volunteered half-formed direction is none of these: it becomes candidate #1 in area H — annotate it `— candidate (user): <one line>` — and must survive the same scoring as every other candidate.

### Drift recovery

The moves and recoveries match the siblings, compressed: `correct` re-answers the node in place. `tangent`/`new-concern` adds a single node inline (a whole new *area* routes through the Facilitator), sets `resume-target` to the shallowest unfinished node (never overwrite an existing pointer — nesting is a hard trigger), pauses the node you're leaving, and descends. On resolution, grade the close, then climb back to `resume-target`; if none, take the next `[unvisited]` in letter order. Most tangents are *abandoned*, not resolved — you won't always notice, which is what the Facilitator is for.

### Dispatching the Scout

The Scout is this lane's workhorse — problem validation is mostly research. Two uses:

- **Opening survey** (single, aimed): at intake, run this **by default whenever the domain plausibly has prior art** — *does something already solve this? what does the prior art look like?* It seeds D and, later, the adopt/buy candidate. Skip it only for a genuinely private or internal problem with nothing to survey — don't manufacture research.
- **Mid-run, on the same rule as ever:** when your recommended answer would otherwise be a load-bearing guess, and to verify a `(claimed)` annotation before it bakes into a `verified` close. Idea-lane queries lean toward problem-evidence, user-behavior, market/prior-art, and internal-data facts.

Pass one specific question or claim plus the scope to look in; fold the finding into your recommendation; log a `meta` row when it materially informed a close, so a verified close reads differently from an assumed one.

### When to call the Facilitator — loose cadence

Hard triggers only — there is no every-N-turns cadence in this lane:

- **Nesting:** a `tangent`/`new-concern` opens while `resume-target` is already set.
- **Deep drift:** you logged `drift: deep` this turn.
- **Sticky tangent:** a tangent survived more than one user turn without resolving or being dropped.
- **Same-node staleness:** `turns-since-node-change ≥ 5`.
- **The commit gate** — mandatory (below).

Soft heuristics: unsure whether the frame is ready to flip phases; a run of `correct` moves on one node; the same pull parked more than twice; a branch that feels finished.

Dispatch `grill-facilitator` with `lane: idea`, the lane dir path, and the stakes + phase. Substantial restructuring — an area-level add, drop, or re-scope — is its call, not yours: hand it your proposal and the live rationale; single-node adds and drops stay inline (logged), reviewed at the gate. There is no Planner in this lane, so for a new area the Facilitator returns the heading and items itself and you transcribe them. Its verdict is binding. Persist every pass: a `meta` row (verdict + climb-back or sanctioned changes) and `last-facilitator: turn N`.

## The direction phase (area H)

**Phase flip.** Set `phase: direction` when every area A–G has its first load-bearing item closed at the stakes bar and B's closure is not `assumed`. The working test: could you *score a candidate* against the frame? If two candidates would tie because the frame can't separate them, the frame isn't ready — stay in `framing`. Premature flips are caught retroactively at the gate; genuine uncertainty here is a soft Facilitator trigger.

**Generate the slate.** 3–5 candidate directions at concept altitude, ≤2 sentences each (strategy + actor + outcome), spanning genuinely different strategies — not five flavors of the same app. Two candidates are mandatory whenever applicable: **do nothing / live with it** (forces the value answer to be honest) and **adopt/buy** (from D's Scout finding; if D never scouted, scout now). The user's half-formed direction, if any, is candidate #1 in their own words — first position, no other privilege. Candidates land as H-area items, so the ordinary map machinery covers them.

**Present once, adjudicate normally.** One slate turn: a table of candidates × the frame — which recorded instances each addresses, fit against F, rough cost against E, the key assumption or risk — with your recommended pick and why. Then return to one-question-at-a-time: corrections, deepening a candidate (Scout on demand), additions. Altitude guard: the moment comparison descends into feature behavior, park the line and lift back to concept.

## The commit gate & completion

The exit is a **user-spoken verdict**, not coverage. Before putting the verdict to the user, run the **mandatory gate pass** — a Facilitator dispatch that in this lane owns both process and substance:

- **process** — drift, solution bleed, the drop review (every `[dropped]` area and node), the premature-flip check;
- **evidence** — unearned conviction: `verified` closes with no instance or Scout row behind them; a load-bearing frame resting on `testimony` at `team`+ stakes;
- **substance** — the **instances-vs-direction walk**: every recorded instance walked against the leading candidate; any instance the direction does not visibly address comes back as a finding.

Surface its findings to the user plainly — including the ones that argue against going. Then ask for the verdict:

- **`go <direction>`** — restate the concept (one paragraph, concept altitude) and the assumptions it rests on (every `assumed` close and flagged testimony that survives); settle the **posture proposal** from E plus the chosen direction.
- **`no-go`** — a success state: the frame, the evidence, and why nothing clears the bar become the brief.
- **`defer`** — name the *specific* evidence that would reopen it; that is the brief's core.
- **`adopt <existing thing>`** — name the residual gap the user accepts.

Then summarize: the verdict; the evidence-grade mix (how much of the frame is `verified` vs `testimony` vs `assumed`); the assumptions carried; the instances recorded; what was parked (count and path); any dropped areas; and, on a `go`, the posture proposal. Finally, **offer** — don't auto-run — `/to-idea` to compile `grill-<slug>/idea.md`; on a `go`, that brief is what `/spec-grill` consumes. The grill produces understanding; converting it is the user's call.

## Templates

**`living-agenda.md`**

```markdown
# Opp-Grill Working Agenda: <topic>

**Position**
- stakes: <casual | team | strategic> — <optional note>
- phase: <framing | direction>
- active: <node id, or —>
- resume-target: <node id, or —>
- turns-since-node-change: 0
- last-facilitator: <turn N, or —>

## A. Stakes & framing — <one line>
- [resolved: verified] A1 <question> — <what it closed to>
- [active] A2 <question>

## B. Evidence & instances — <one line>
- [resolved: verified] B1 <question> — instance: <the concrete occurrence>
- [unvisited] B2 <question> — given (claimed): <world-fact to verify>

## F. Constraints & non-negotiables — <one line>
- [unvisited] F1 <question> — given (decided): <one line>

## H. Direction — candidates, selection, the gate
- [unvisited] H1 <candidate, concept altitude> — candidate (user): <one line>
- [unvisited] H2 <candidate, concept altitude>
```

**`conversation-path.md`**

```markdown
# Opp-Grill Conversation Path: <topic>

| turn | active-node | move | summary | implication | drift | next-node |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | A1 | answer | stakes confirmed: team | instance or scout on load-bearing claims | none | A2 |
| 3 | B1 | answer | instance: 2026-06 release slipped on triage | B1 verified | none | B2 |
| 4 | C1 | park | parked: proposed auto-reply feature | product question re-asked | none | C1 |
| 6 | D2 | meta | scout: two SaaS tools cover triage partially (web) | adopt/buy is a live candidate | none | D2 |
| 9 | H3 | meta | facilitator gate: instance 2 unaddressed by H2 | put finding to user before verdict | none | H2 |
```

**`product-parking-lot.md`**

```markdown
# Product Parking Lot: <topic>

Append-only. One line per item. Nothing here binds this lane — /to-idea carries these forward as
non-binding notes, and spec-grill's intake seeds from them. Technical pulls carry [technical].

- <one-line product pull> (turn N, near <node-id>)
- <one-line product detail the user has settled> (intake) [decided]
- <one-line technical pull> (turn N, near <node-id>) [technical]
```
