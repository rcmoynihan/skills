---
name: to-idea
description: Formalize a completed opp-grill session (or the current conversation) into a single idea brief markdown file in the grill run dir — the problem/opportunity frame with its evidence grades, the considered directions, and the commit verdict (go / no-go / defer / adopt). Deliberately half a spec; on a go, it is the input /spec-grill consumes. Synthesis only — no re-interview.
argument-hint: "(optional) path to a grill run dir; blank uses the most recent run + this conversation"
disable-model-invocation: true
---

# to-idea

Turn everything an opportunity grilling settled — the frame, the evidence, the candidates, the verdict — into one durable **idea brief**. This is the conversion step opp-grill's completion offers: the grill produced *understanding*; this produces the *document*. The brief must stand alone: `/spec-grill` — or a brand-new agent or developer — picks up the run from this file without ever seeing the interview behind it. A `no-go` or `defer` brief is a complete, successful artifact: the durable record of why not, so the next person who feels this pain finds the investigation instead of redoing it.

**Synthesize, do not interview.** The grill already did the asking. Build the brief from what was actually decided and observed — never re-open the interview, and never invent an answer the grill didn't reach. Where the grill left a hole, the brief says so.

**Deliberately half a spec.** The brief carries no coded-ID system — no `INV-`/`REQ-` analog — on purpose: it is the *input* to the product interview, not a contract, and IDs would tempt `/spec-grill` to hold it as locked. Its authority travels as conviction-tagged constraints and evidence grades, which spec-grill's intake picks up as givens.

## The boundary: what stays here vs goes to the spec

The brief owns the problem space plus one decision at concept altitude. The test for any piece of content: **would it change if we solved the same problem with a different product?** No → it belongs here. Yes → it is spec-lane material, and the brief carries it only as a non-binding Carried-Forward Note. The one exception is §8's chosen direction, stated at concept altitude ("a Slack bot that auto-triages inbound tickets") — never features, screens, or behaviors.

## How to run this

- **Never invent.** Every statement traces to something the grill settled, the Scout found, or the user said. An inference needed for coherence is tagged `(assumed — confirm)` — never stated as settled.
- **Preserve the evidence grades.** Every claim in §2 keeps its grade (`verified` / `testimony` / `assumed`) and, where one exists, its recorded instance or source. An all-testimony frame says so plainly near the top — the grade mix is information, not embarrassment.
- **Honest about gaps.** Unvisited, paused, and dropped concerns become entries in §9 with their reason; don't paper over an untouched area.
- **Self-contained for a cold reader.** On a `go`, the brief is spec-grill's primary input, so it carries its own context: the actors, the status quo, the instances, the stakes.
- **No change narration.** The brief describes the opportunity and the decision as they stand, for a reader who never saw the grill. On a re-run, overwrite so it reads as written fresh.
- **The file is the deliverable.** Write it, then return only the absolute path plus a 1–2 line summary — never paste the brief body into chat.

## Find the input

A finished opp-grill leaves its state in the chain's run dir. Rediscover the most recent run:

```bash
ls -dt "${TMPDIR:-/tmp}"/code-goblin-pro/grill-*/ 2>/dev/null
```

If the argument names a run dir, use that. If a recent run matches the topic, read the idea lane's three files — your primary source:

- `idea/living-agenda.md` — item statuses with evidence grades, node annotations (givens, claims, candidates), and the Position block (`stakes`, `phase`).
- `idea/conversation-path.md` — the turn-by-turn trail: the actual answers, the Scout findings (`meta` rows), the gate pass, and the verdict turn.
- `idea/product-parking-lot.md` — the deflected pulls; the sole source of §10.

If `idea.md` already exists this is a recompile — overwrite it, and if `spec.md` (or later artifacts) exist downstream, warn that they may now be stale.

Also draw on this conversation. **Standalone fallback:** if no run dir matches, synthesize from the conversation alone — don't require a grill. Create `grill-<slug>/` (slug from the topic) and write the brief there so `/spec-grill` can find it later; write no lane files. In this mode the verdict is whatever the user actually spoke; absent one, use `verdict: undecided` and note in §8 that the brief was compiled without a gate.

## Process

1. **Locate & read** the grill run (above), or fall back to the conversation.
2. **Extract the verdict and the frame's non-negotiables first:** the verdict turn (or `undecided`), the stakes, every constraint with its conviction tag, and the evidence-grade mix. These govern the document — a brief whose §8 contradicts its §6 was compiled wrong.
3. **Route the lane's material:** `[resolved]` items → their sections, grades and instances preserved; `[unvisited]` / `[paused]` / `[dropped]` → §9 with their reason; every `assumed` close and flagged testimony → §9; the parking lot → §10, verbatim-in-spirit, `[technical]` and conviction tags intact.
4. **Draft the sections in order** (template below). Fill what applies; a section with nothing gets a one-line `None.` — do not pad.
5. **Consistency pass.** §8's direction must honor every §6 constraint; every §9 assumption traces to a close or a gap; then the cold-reader check: could someone who never saw the conversation run `/spec-grill` from this file alone?
6. **Save & report.** Write to `grill-<slug>/idea.md`. Return the absolute path + a 1–2 line summary: topic, verdict, stakes, the grade mix, # assumptions carried, # carried-forward notes. On a `go`, note that `/spec-grill` is the next link when the user wants the product side defined.

## Brief template

<brief-template>
```markdown
---
title: <concise name for the opportunity>
status: draft
date_created: <YYYY-MM-DD>
source: <grill run dir path, or "conversation">
stakes: <casual | team | strategic> — <optional note>
verdict: <go <direction, one line> | no-go | defer | adopt <tool> | undecided>
posture_proposal: <poc | internal-tool | product-feature | new-system> — <note>   # go only; omit otherwise
tags: [<relevant tags>]
---

## 1. Problem / Opportunity Statement

One paragraph, from the affected actor's perspective: what is happening (or what just became
possible) and why it matters now.

## 2. Evidence

The concrete instances and findings the frame rests on, each with its grade — `verified` /
`testimony` / `assumed` — and its instance or source. State the mix plainly if it leans on
testimony.

## 3. Actors & Pain

Who specifically is affected, what it costs each, how often.

## 4. Status Quo & Prior Art

What happens today, why the workarounds fail, and what already exists that addresses this
(with sources when the Scout found them).

## 5. Value & Appetite

What solving this is worth, the cost of doing nothing, and the time/money the user would
actually spend.

## 6. Constraints & Non-Negotiables

What any solution must respect — one line each, conviction-tagged `[decided|leaning]`. These
become spec-grill givens.

## 7. Outcome Measures

How success would be observed, outcome-level only — never acceptance criteria.

## 8. Direction & Verdict

The verdict, then — on a `go` — the chosen concept in ≤1 paragraph at concept altitude, and why
it won. Then **Alternatives considered**: every other candidate, one line each with why it was
rejected (including do-nothing and adopt/buy). On `no-go`/`defer`/`adopt`: why nothing cleared
the bar, the evidence that would reopen it, or the adopted tool and the residual gap.

## 9. Assumptions & Open Risks

Every `assumed` close and load-bearing testimony the verdict rests on; the concerns the grill
never reached, each with its reason. On a `defer`, the reopening evidence leads this section.

## 10. Carried-Forward Notes

Non-binding. The parked pulls, one line each, deduplicated, origin and `[technical]` /
conviction tags preserved verbatim — spec-grill's intake seeds from these. `None.` if the lot
is empty.

## 11. Related

Paths to the idea lane's artifacts (living-agenda / conversation-path / product-parking-lot)
and the downstream `grill-<slug>/spec.md` once produced.
```
</brief-template>
