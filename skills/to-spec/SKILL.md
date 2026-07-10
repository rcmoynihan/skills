---
name: to-spec
description: Formalize a completed spec-grill session (or the current conversation) into a single product PRD/spec markdown file in the grill run dir — the *what*: product framing (problem, solution, scope) and the governing invariants, followed by a rigorous, testable statement of user stories, requirements, external-interface promises, acceptance criteria, and validation. Synthesis only — no re-interview. The technical *how* is settled by /design-grill and compiled by its companion /to-design doc.
argument-hint: "(optional) path to a grill run dir; blank uses the most recent run + this conversation"
disable-model-invocation: true
---

# to-spec

Turn everything a spec grilling settled — the decisions, clarifications, and constraints — into one durable spec. This is the conversion step spec-grill's completion offers: the grill produced *understanding*; this produces the *document*. The document must then **stand alone**: the next link in the chain (`/design-grill`) — or a brand-new agent or developer — picks up the run from this file without ever seeing the grill conversation behind it.

**Synthesize, do not interview.** The grill already did the asking. Build the spec from what was actually decided and from the conversation — never re-open the interview, and never invent an answer the grill didn't reach. Where the grill left a concern unexplored, the spec says so; it does not fill the hole with a guess.

The output is a single markdown file written to the run dir: product framing (problem, solution, scope) followed by the **governing invariants**, plus a rigorous, testable statement of *what* the system must do and be (user stories, coded requirements, external-interface promises, acceptance criteria, validation). It is the *what*; the technical *how* is grilled by `/design-grill` and compiled into its companion `/to-design` doc.

## The boundary: what stays here vs goes to the design doc

`to-spec` owns the *what* — the product intent and the behavior a reader can observe. The technical *how* — architecture, data/state, interface realization, operations, deployment, technical decisions, and test seams — is the companion `/to-design` doc. The test for any piece of content: **would it change if we re-implemented the same product a different way?** No → it stays here. Yes → it belongs in the design doc, and this spec only references it.

So behavioral acceptance criteria, the external-interface *promise*, NFR *targets*, and the invariants stay here — invariants as the source of truth, carried forward into the design lane by id. Their *realization* — schemas, mechanisms, seams, ADRs — lives in the design doc. Technical asides the grill deflected sit in exactly one place: the non-binding **Carried-Forward Technical Notes** section (§12), which seeds the design grill's agenda and carries no spec authority — a line's `[decided|leaning]` conviction tag travels onward as a design-lane given.

## How to run this

- **Lock the invariants before the details.** Frame the problem and solution first (sections 1–2), then write the `## 3. Invariants & Non-Negotiables` block before drafting anything from section 4 onward. Reasoning the non-negotiables before the requirements, stories, and decisions is what keeps every one of them consistent — they all sit below the invariants and must honor them.
- **Never invent.** Every requirement, story, and acceptance criterion traces to something the grill settled or the user said. An inference you must make to keep the spec coherent is allowed only if tagged `(assumed — confirm)` — never stated as a decided requirement. Parked technical lines are *not* material for requirements or invariants — they land in §12 verbatim-in-spirit and nowhere else.
- **Honest about gaps.** A grill that covered little yields a *partial* spec, and that is correct. Unexplored/paused concerns become Open Questions; abandoned ones become Out of Scope. Don't paper over an untouched area.
- **Self-contained for a cold reader.** The spec is the design grill's only required input, so it carries its own context: who the actors are (§4), what the world looks like today and why this is being built (§2), the primary journeys end to end (§5), and the run's posture (frontmatter). If a section leans on something only the conversation knew, the spec is not done.
- **No volatile detail.** Describe the *what*, not the *how*: no specific file paths, no code snippets (one exception — a snippet that encodes a decision more precisely than prose, e.g. a schema, type shape, or state machine; trim to the decision-rich part).
- **No change narration.** The spec describes the system as it is to be built, for a reader who never saw the grill. No "we decided to change…", no process notes.
- **The file is the deliverable.** Write it, then return only the absolute path plus a 1–2 line summary — never paste the spec body into chat.

## Find the input

A finished spec grill leaves its state in the chain's run dir. Rediscover the most recent run:

```bash
ls -dt "${TMPDIR:-/tmp}"/code-goblin-pro/grill-*/ 2>/dev/null
```

If the argument names a run dir, use that. If a recent run matches the topic at hand, read the spec lane's five files — they are your primary source:

- `spec/initial-agenda.md` — the full concern map; your coverage baseline (what *should* have been examined).
- `spec/living-agenda.md` — item statuses (`[resolved]` / `[paused]` / `[dropped]` / `[unvisited]`) and the Position block (including the `posture`); this is the routing key for which concern goes to which section.
- `spec/conversation-path.md` — the turn-by-turn decision trail; this is where the actual decisions and their implications live.
- `spec/technical-parking-lot.md` — the technical asides the grill deflected; the sole source of §12.
- `spec/givens.md` — the a-priori details the user brought, with conviction and disposition; a `[consumed]` `decided` given is prime invariant material.

**Brief awareness:** when the run dir also holds `idea.md` (the chain started at `/opp-grill`), read it too — source material for §§1–2 framing and §9 rationale, attributed `Source: brief`. Where the spec lane's own decisions diverge from it, the lane wins: the grill grilled the brief's frame.

**Amendment awareness:** on a re-run of a run whose `design/living-agenda.md` exists, read its Locked block — every entry tagged `(amended)` is the *current* product decision, superseding what the spec lane recorded; fold each one in (its `amend` row in `design/conversation-path.md` is the source).

Also draw on this conversation. **Standalone fallback:** if no run dir matches, synthesize from the conversation alone — don't require a grill — and still produce the governing invariants block. In this mode, **create** `grill-<slug>/` (slug from the topic) and write the spec there: the run dir anchors the chain so `/design-grill` can find the spec later.

## Process

1. **Locate & read** the grill run (above), or fall back to the conversation.
2. **Extract the non-negotiables early.** Pull every non-negotiable: hard constraints, decisions the user insisted on, locked tradeoffs, anything stated as a must/never. A `[consumed]` `decided` given is non-negotiable material of the first order — attribute it `Source: given (intake)`, `given (turn N)`, or `given (brief)`. `leaning` givens that shaped resolutions are ordinary decision material; for a `[superseded]` given the superseding resolution is the material, not the given; `[set-aside]` givens compile to nothing (Out of Scope only if the user said so). These become the `## 3. Invariants & Non-Negotiables` block and govern every section below them — settle them before drafting the detailed requirements, stories, and decisions, even though the problem and solution framing (sections 1–2) is written above them.
3. **Route the grill's items:** `[resolved]` + the decision trail → the body sections; `[unvisited]`/`[paused]` → **Open Questions**; `[dropped]` → **Out of Scope**; the parking lot → **Carried-Forward Technical Notes**. Speak the grill's own vocabulary.
4. **Draft the sections in order** (template below), each grounded in the artifacts/conversation and consistent with the invariants; drop the non-negotiables you extracted in step 2 into section 3. Fill what applies; a section with nothing gets a one-line `None.` — do not pad.
5. **Consistency pass.** Re-read the invariants block and check the whole document against it: no requirement, user story, acceptance criterion, or decision may contradict an `INV-###`. Fix any that do before saving. Then run the cold-reader check: could someone who never saw the conversation run the design grill from this file alone?
6. **Save & report.** Write to `grill-<slug>/spec.md` in the run dir. Return the absolute path + a 1–2 line summary: topic, # invariants, # requirements, # open questions, # carried-forward notes, and the word **partial** if material concern areas are still unresolved. Note that `/prior-art-check` is the recommended next step — it checks whether an existing solution already satisfies this spec before the design phase is grilled — and `/design-grill` follows when the user wants the technical side settled.

## Spec template

Write the file in this shape. Coded IDs (`INV-`, `REQ-`, `CON-`, `GUD-`, `AC-`) make the spec trackable and testable; keep them sequential within each prefix.

<spec-template>
```markdown
---
title: <concise title for what's being built>
status: draft
date_created: <YYYY-MM-DD>
source: <grill run dir path, or "conversation">
posture: <poc | internal-tool | product-feature | new-system> — <optional note>
tags: [<relevant tags>]
---

## 1. Problem Statement

The problem being solved, from the user's perspective.

## 2. Solution Overview, Context & Scope

The solution from the user's perspective; the context a cold reader needs — what exists today, why this is being built now, and the environment it lands in; the intended audience and key assumptions; what is in scope and, briefly, what is not.

## 3. Invariants & Non-Negotiables

The constraints the rest of the spec must honor. They outrank every requirement, story, and decision below them.

- **INV-001**: <canonical one-line non-negotiable>
    - Does NOT mean: <the tempting misreading this rules out>   ← only for misread-prone ones
    - Source: <grill turn/node, e.g. "turn 4 (A4/E1)", "given (intake)", or "assumed — confirm">
- **INV-002**: <…>

## 4. Definitions & Actors

Domain terms, acronyms, and the actors/personas used below, so the spec is self-contained — every actor named in a story or journey is defined here.

## 5. User Journeys & Stories

First the 2–4 **primary journeys**, each narrated end to end: the actor, where they start, what they do, what they observe at each step, where it ends. Then an extensive, numbered story list — `As an <actor>, I want <feature>, so that <benefit>` — covering every facet the grill touched.

## 6. Requirements & Constraints

Explicit, testable statements. A constraint that restates an invariant references its id. State NFR *targets* here (e.g. a latency or availability bound); the *mechanism* that meets them lives in the design doc.

- **REQ-001**: <requirement>
- **CON-001**: <constraint>  (e.g. "honors INV-001")
- **GUD-001**: <guideline / recommendation>

## 7. External Interfaces & Contracts

The interface *promise*, at the behavioral level: which operations / commands / events exist, what each one means, the inputs a consumer must supply and the outcomes it can rely on, and the *meaning* of errors. Wire schemas, field types, error codes, versioning mechanics, and idempotency detail are *realization* — they live in the design doc. Include a shape here only when it's a frozen contract external consumers integrate against.

## 8. Acceptance Criteria

Testable, in Given-When-Then form, tied back to requirement ids. These behavioral promises are what the design doc's test strategy verifies against.

- **AC-001**: Given <context>, when <action>, then <expected outcome>.  (REQ-001)

## 9. Key Decisions & Rationale

The product and scope decisions the grill produced and *why* — what to build, for whom, what to include or defer, interaction models. **Technical/architectural decisions** — a choice with a rejected technical alternative (storage, schema, technology, algorithm) — belong to the design grill and its doc as `DD-###` ADRs, not here.

## 10. Dependencies & External Integrations

External systems, services, platforms, data, and compliance needs — *what* is required, not specific package versions. The integration *design* (clients, pooling, failover) lives in the design doc.

## 11. Open Questions & Unresolved Decisions

What the grill did not settle. Each tagged with its grill node id (e.g. `C2`, `F3`). Never answered here — these are the calls still owed. If this section is large, note near the top that the spec is **partial**.

## 12. Carried-Forward Technical Notes

Non-binding on this spec. Technical asides deflected during the product interview — inputs to the design grill's agenda. Nothing here carries spec authority. One line each, deduplicated, with origin (turn/node or intake) and any `[decided|leaning]` conviction tag preserved verbatim — a tagged line is the user's stated conviction, and the design grill picks it up as a given. `None.` if the parking lot is empty.

## 13. Out of Scope

Explicit non-goals, including concerns the grill deliberately dropped.

## 14. Validation Criteria

How to verify a built solution meets this spec's *what* — the product-level, behavioral checks. Technical verification (test seams, load/chaos, readiness) lives in the design doc.

## 15. Related / Further Reading

Paths to the spec lane's artifacts (initial-agenda / living-agenda / conversation-path / technical-parking-lot / givens), the companion `grill-<slug>/design.md` once produced, and any other relevant docs.
```
</spec-template>
