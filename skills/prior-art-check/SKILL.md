---
name: prior-art-check
description: Check whether an existing solution already satisfies a compiled spec before the design phase is grilled — the spec-keyed buy-vs-build gate. Reads spec.md, surveys candidate solutions (internal, open-source, commercial) via the Scout, scores each against the spec's invariants, load-bearing requirements, and primary journeys in a coverage matrix, and ends at a commit gate whose verdict is the user's: build / adopt / adopt-with-gaps / defer. Research only — no re-interview; it never re-opens the spec or rules the idea dead on its own. The gate between /to-spec and /design-grill in the opp-grill → spec-grill → design-grill → to-design chain.
argument-hint: "(optional) path to a grill run dir or spec file; blank uses the most recent run"
disable-model-invocation: true
---

# Prior-Art-Check

This gate serves the seam between a finished spec and the design grill. The spec now states, crisply, what must be true — its invariants, requirements, journeys, and acceptance criteria — and before that crisp statement of needs is spent grilling *how* to build it, this step asks whether something already **is** it. The failure mode it exists to catch: running the whole chain and only afterward noticing that an existing tool, library, or product already covers the need — realized too late, after the most expensive phase.

`/opp-grill` asks the same question upstream, but at **concept altitude** — "does anything address this problem?" — before any requirement exists, so its match is necessarily coarse. Here the needs are concrete `INV-`/`REQ-`/`AC-` criteria, so the match can be too: not "is there a triage tool," but "does *this* triage tool satisfy *these* invariants and journeys." That fine-grained comparison is only possible after `/to-spec`, which is why the gate lives here and not earlier.

Like opp-grill's commit gate, it ends at a **user-spoken verdict**. The check surfaces the terrain and recommends; it never rules the idea dead on its own. Prior art *informs* the build-vs-adopt call — it does not pre-answer it.

## What this is — and what it is not

**You run the Scout and hold the matrix. You do not interview, design, or re-open the spec.** The spec is fixed input: its invariants and requirements are the criteria a candidate is scored against, not decisions to relitigate. This is a mostly non-interactive research pass with a single decision turn at the end — not a grill. There is no living agenda, no per-turn log, no lane subdir; the deliverable is one artifact plus a verdict.

Two boundaries:

- **The spec is locked.** If a candidate looks compelling only because it ignores an `INV-###`, that is a *finding for the user* — "adopting X means relaxing INV-002; that's a spec-lane call" — never a criterion you quietly drop. You evaluate against the spec as written.
- **The specification floor.** You resolve exactly one thing — the build-vs-adopt verdict — at the same altitude opp-grill resolves *which direction*: concept, not feature behavior. You do not evaluate *how* a candidate is built or design an integration; that is the design grill's job on a `build` outcome.

## Run directory & input

The whole chain lives in one run dir in the system temp dir. Discover it — never re-derive a slug:

```bash
ls -dt "${TMPDIR:-/tmp}"/code-goblin-pro/grill-*/ 2>/dev/null
```

If the argument names a run dir or a spec file, use that; otherwise take the most recent run matching the topic. Place the run:

- **`spec.md` is the required input.** No `spec.md` in any matching run → **decline and route**: `/to-spec` to compile a spec from a finished spec grill or the conversation, or `/spec-grill` to grill one first. There is nothing to match a candidate against without it.
- **`idea.md`, when present** (the chain started at `/opp-grill`): read its §4 *Status Quo & Prior Art* and its verdict — the opening survey's findings seed your candidate list, and an upstream `adopt` verdict is a strong prior you are now testing against the concrete spec.
- **`prior-art.md` already exists** — this gate already ran. A new run is a **re-check**: state the previous verdict, warn it goes stale, and confirm before proceeding (a materially changed spec is the usual reason to re-run).

The gate writes exactly one file, `grill-<slug>/prior-art.md`, at the run-dir top level alongside `idea.md` / `spec.md` / `design.md`. It creates no lane subdir.

## How to run this

- **One subagent, by name:** `grill-scout` (`grill-scout`), dispatched ad-hoc for research — the same worker opp-grill and the grills use. Use the Agent tool (`subagent_type`); if `/agents` shows it plugin-scoped, use that form. No Planner, Facilitator, or Tracer in this gate.
- **Synthesize, don't interview.** Derive the criteria and candidates from the spec and the Scout's briefs. The only turn you take from the user is the verdict.
- **The file is the deliverable.** Write `prior-art.md`, then return only its absolute path plus a 1–2 line summary — never paste the body into chat.

### Process

1. **Derive the match criteria from `spec.md`.** These are the matrix's columns — the criteria that would actually *separate* candidates, not every id: the **invariants** (`INV-###`, the disqualifiers), the **load-bearing requirements** (`REQ-###` that carry the product's core value), and the **primary journeys** (§5, end to end). Pull in a `CON-###` or a §10 dependency when deployment/integration fit is itself decisive. Keep the set tight — a dozen criteria that discriminate, not fifty that don't.

2. **Assemble the candidate list.** Seed from `idea.md`'s prior-art section (if present), then dispatch the Scout to find the rest across surfaces — org-internal systems (via `gh`), open-source libraries, and commercial/SaaS products. Include the honest floor candidates: **build from scratch** (the baseline every candidate is measured against) and, where relevant, **compose existing internal primitives**. If the domain genuinely has no prior art, say so in one line and record a short artifact recommending `build` — don't manufacture candidates.

3. **Score each candidate into a coverage matrix.** Candidates are rows, criteria are columns; each cell is `✅ full` / `🟡 partial` / `❌ none` / `❓ unknown` (the Scout couldn't confirm — never guess a ✅). For each candidate also record: **adoption/switching cost**, **licensing / pricing / hosting**, **maturity & risk**, and the **residual gap** — what you would still have to build on top. A candidate that fails an `INV-###` is marked disqualified with the invariant named, not silently zeroed.

4. **Assess and recommend.** Rank candidates by coverage-and-fit against the criteria net of adoption cost. Name the strongest adopt candidate and its residual gap versus building fresh, and state your recommended verdict with the reasoning. Be adversarial toward building: the default regret this gate exists to prevent is *building something that already exists*.

5. **Put the verdict to the user** (below). Present the matrix and recommendation plainly — including the case against your own recommendation — then ask.

6. **Write `prior-art.md`** (template below) and report the path.

## The verdict

The exit is a **user-spoken verdict**, not a score. It mirrors opp-grill's commit gate:

- **`build`** — nothing clears the bar; proceed to `/design-grill`. Record *why each serious candidate fell short* against the criteria — this **differentiation rationale** is what the design grill must honor so it doesn't reinvent what a candidate already does well, and knows exactly where the product earns its existence.
- **`adopt <thing>`** — a candidate satisfies the needs; name the **residual gap** the user accepts. This ends the chain the way a `no-go` does: a documented "we looked, and adopting X is the right call" is this gate working as intended, not a failure.
- **`adopt-with-gaps <thing>`** — adopt a candidate as the base and build only the residual gap on top. Record the gap concretely and recommend the user **re-scope the spec to it** (the reduced spec is what `/design-grill` then grills); the design grill treats the adopted base as a locked dependency/given. Keep this a recommendation — re-scoping the spec is a spec-lane action, not something this gate performs.
- **`defer`** — the match can't be settled yet; name the *specific* research or trial (a spike against candidate X, a pricing answer, a missing capability confirmation) that would settle it. That named next step is the artifact's core.

## Dispatching the Scout

The Scout is this gate's only researcher. Dispatch `grill-scout` with a **targeted query** and the scope to look in; parallelize by surface when it sharpens the survey — one pass for org-internal (`gh`), one for open-source, one for commercial — rather than one vague sweep. Frame each query concretely against the spec's criteria: not "what triage tools exist," but "does <candidate> support <these journeys> and honor <these invariants>; what does it cost to adopt; what does it not do." Name the lane as `spec` and pass the spec path (`grill-<slug>/spec.md`) — the Scout can't see this conversation.

Fold each brief into the matrix. The Scout **reports terrain and does not rule on the route** — a candidate's coverage is a fact it finds; whether that coverage is *enough* is your assessment, and whether to adopt is the user's verdict. A `❓ unknown` cell is an honest output; it may become a `defer` if it's decisive.

## The artifact — `prior-art.md`

<prior-art-template>
```markdown
---
title: Prior-Art Check — <topic>
date: <YYYY-MM-DD>
source: <grill run dir path>
spec: <path to spec.md>
verdict: <build | adopt <thing> | adopt-with-gaps <thing> | defer>
---

## 1. Match Criteria

The spec criteria a candidate was scored against — the ones that discriminate.

- **INV-002** — <one line>  *(disqualifier)*
- **REQ-004** — <one line>
- **Journey: <name>** — <one line, end to end>

## 2. Candidates Surveyed

One block per candidate: category (internal / OSS / commercial / build), source or link, maturity, licensing / pricing / hosting.

## 3. Coverage Matrix

| Candidate | INV-002 | REQ-004 | Journey X | … | Adoption cost | Residual gap |
| --- | --- | --- | --- | --- | --- | --- |
| Build from scratch | ✅ | ✅ | ✅ | … | — (baseline) | none |
| <Tool A> | ✅ | 🟡 | ❌ | … | <low/med/high + note> | <what's left to build> |
| <Tool B> | ❌ *(fails INV-002)* | ✅ | ✅ | … | <…> | <…> |

Cells: ✅ full · 🟡 partial · ❌ none · ❓ unknown (unconfirmed).

## 4. Assessment & Recommendation

The ranking and the reasoning; the strongest adopt candidate and its residual gap versus building; the recommended verdict and the honest case against it.

## 5. Verdict & Residual Gap

The user's verdict, verbatim in intent, and — for `adopt` / `adopt-with-gaps` — the residual gap accepted; for `defer` — the specific research that would settle it.

## 6. Differentiation Rationale   *(on a `build` verdict)*

Why each serious candidate fell short against the criteria — the guidance the design grill honors so it neither reinvents a candidate's strengths nor loses the reasons the product exists.

## 7. Related

Path to `spec.md`, to `idea.md` (if present), and any load-bearing sources the Scout cited.
```
</prior-art-template>

## Completion & hand-off

Summarize for the user: the verdict, the candidate field and how the leader scored, the residual gap (or the deferred question), and where the chain goes next. Then **offer** — don't auto-run — the next step:

- On **`build`** — `/design-grill` grills the technical side against the spec; it will read `prior-art.md`'s differentiation rationale.
- On **`adopt`** — the chain is complete; the artifact is the record of why.
- On **`adopt-with-gaps`** — the user re-scopes the spec to the residual gap (edit `spec.md` or re-grill the reduced scope), then `/design-grill` grills that.
- On **`defer`** — run the named research or spike, then re-run this gate.

The gate produces a decision; acting on it is the user's call.
