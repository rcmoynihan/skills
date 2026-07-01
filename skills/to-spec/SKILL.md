---
name: to-spec
description: Formalize a completed grillmaster session (or the current conversation) into a single product PRD/spec markdown file in the system temp dir — the *what*: invariants first, then product framing (problem, solution, user stories, scope) and a rigorous, testable statement of requirements, external-interface promises, acceptance criteria, and validation. Synthesis only — no re-interview. The technical *how* is its companion `/to-design` doc.
argument-hint: "(optional) path to a grillmaster run dir; blank uses the most recent run + this conversation"
disable-model-invocation: true
---

# to-spec

Turn everything a grilling settled — the decisions, clarifications, and constraints — into one durable spec. This is the conversion step Grillmaster's completion offers: the grill produced *understanding*; this produces the *document*.

**Synthesize, do not interview.** The grill already did the asking. Build the spec from what was actually decided and from the conversation — never re-open the interview, and never invent an answer the grill didn't reach. Where the grill left a concern unexplored, the spec says so; it does not fill the hole with a guess.

The output is a single markdown file written to the system temp dir: product framing (problem, solution, user stories, scope) plus a rigorous, testable statement of *what* the system must do and be (coded requirements, external-interface promises, acceptance criteria, validation) — and it **leads with the invariants**. It is the *what*; the technical *how* is its companion `/to-design` doc.

## The boundary: what stays here vs goes to the design doc

`to-spec` owns the *what* — the product intent and the behavior a reader can observe. The technical *how* — architecture, data/state, interface realization, operations, deployment, technical decisions, and test seams — is the companion `/to-design` doc. The test for any piece of content: **would it change if we re-implemented the same product a different way?** No → it stays here. Yes → it belongs in the design doc, and this spec only references it.

So behavioral acceptance criteria, the external-interface *promise*, NFR *targets*, and the invariants stay here — invariants as the source of truth, carried forward into the design doc by id. Their *realization* — schemas, mechanisms, seams, ADRs — lives in the design doc.

## How to run this

- **Invariants first, literally.** Before drafting any other section, write the `## Invariants & Non-Negotiables` block at the top of the file. Reasoning the non-negotiables first is what keeps every requirement, story, and decision below them consistent. Do not start section 1 until the invariants block exists.
- **Never invent.** Every requirement, story, and acceptance criterion traces to something the grill settled or the user said. An inference you must make to keep the spec coherent is allowed only if tagged `(assumed — confirm)` — never stated as a decided requirement.
- **Honest about gaps.** A grill that covered little yields a *partial* spec, and that is correct. Unexplored/paused concerns become Open Questions; abandoned ones become Out of Scope. Don't paper over an untouched area.
- **No volatile detail.** Describe the *what*, not the *how*: no specific file paths, no code snippets (one exception — a snippet that encodes a decision more precisely than prose, e.g. a schema, type shape, or state machine; trim to the decision-rich part).
- **No change narration.** The spec describes the system as it is to be built, for a reader who never saw the grill. No "we decided to change…", no process notes.
- **The file is the deliverable.** Write it, then return only the absolute path plus a 1–2 line summary — never paste the spec body into chat.

## Find the input

A finished grill leaves its state in the system temp dir. Rediscover the most recent run:

```bash
ls -dt "${TMPDIR:-/tmp}"/grillmaster-*/ 2>/dev/null
```

If the argument names a run dir, use that. If a recent run matches the topic at hand, read all three files — they are your primary source:

- `initial-agenda.md` — the full concern map; your coverage baseline (what *should* have been examined).
- `living-agenda.md` — item statuses (`[resolved]` / `[paused]` / `[dropped]` / `[unvisited]`) and the Position block; this is the routing key for which concern goes to which section.
- `conversation-path.md` — the turn-by-turn decision trail; this is where the actual decisions and their implications live.

Also draw on this conversation. **Standalone fallback:** if no run dir matches, synthesize from the conversation alone — don't require a grill — and still lead with the invariants block.

## Process

1. **Locate & read** the grill run (above), or fall back to the conversation.
2. **Extract and write the invariants block first.** Pull every non-negotiable: hard constraints, decisions the user insisted on, locked tradeoffs, anything stated as a must/never. Write `## Invariants & Non-Negotiables` at the top of the file before anything else.
3. **Route the grill's items:** `[resolved]` + the decision trail → the body sections; `[unvisited]`/`[paused]` → **Open Questions**; `[dropped]` → **Out of Scope**. Speak the grill's own vocabulary.
4. **Draft the remaining sections in order** (template below), each grounded in the artifacts/conversation and consistent with the invariants. Fill what applies; a section with nothing gets a one-line `None.` — do not pad.
5. **Consistency pass.** Re-read the invariants block and check the whole document against it: no requirement, user story, acceptance criterion, or decision may contradict an `INV-###`. Fix any that do before saving.
6. **Save & report.** Write to `${TMPDIR:-/tmp}/spec-<slug>.md` (`<slug>` from the grill run name or the topic). Return the absolute path + a 1–2 line summary: topic, # invariants, # requirements, # open questions, and the word **partial** if material concern areas are still unresolved.

## Spec template

Write the file in this shape. Coded IDs (`INV-`, `REQ-`, `CON-`, `GUD-`, `AC-`) make the spec trackable and testable; keep them sequential within each prefix.

<spec-template>
```markdown
---
title: <concise title for what's being built>
status: draft
date_created: <YYYY-MM-DD>
source: <grillmaster run dir path, or "conversation">
tags: [<relevant tags>]
---

## Invariants & Non-Negotiables

The constraints everything below must honor. Written first; they outrank every requirement and decision in this spec.

- **INV-001**: <canonical one-line non-negotiable>
    - Does NOT mean: <the tempting misreading this rules out>   ← only for misread-prone ones
    - Source: <grill turn/node, e.g. "turn 4 (A4/E1)", or "assumed — confirm">
- **INV-002**: <…>

## 1. Problem Statement

The problem being solved, from the user's perspective.

## 2. Solution Overview & Scope

The solution from the user's perspective; the intended audience and key assumptions; what is in scope and, briefly, what is not.

## 3. Definitions

Domain terms and acronyms used below, so the spec is self-contained.

## 4. User Stories

An extensive, numbered list. Each: `As an <actor>, I want <feature>, so that <benefit>`. Cover every facet the grill touched.

## 5. Requirements & Constraints

Explicit, testable statements. A constraint that restates an invariant references its id. State NFR *targets* here (e.g. a latency or availability bound); the *mechanism* that meets them lives in the design doc.

- **REQ-001**: <requirement>
- **CON-001**: <constraint>  (e.g. "honors INV-001")
- **GUD-001**: <guideline / recommendation>

## 6. External Interfaces & Contracts

The interface *promise*, at the behavioral level: which operations / commands / events exist, what each one means, the inputs a consumer must supply and the outcomes it can rely on, and the *meaning* of errors. Wire schemas, field types, error codes, versioning mechanics, and idempotency detail are *realization* — they live in the design doc. Include a shape here only when it's a frozen contract external consumers integrate against.

## 7. Acceptance Criteria

Testable, in Given-When-Then form, tied back to requirement ids. These behavioral promises are what the design doc's test strategy verifies against.

- **AC-001**: Given <context>, when <action>, then <expected outcome>.  (REQ-001)

## 8. Key Decisions & Rationale

The product and scope decisions the grill produced and *why* — what to build, for whom, what to include or defer, interaction models. **Technical/architectural decisions** — a choice with a rejected technical alternative (storage, schema, technology, algorithm) — belong in the design doc as `DD-###` ADRs, not here.

## 9. Dependencies & External Integrations

External systems, services, platforms, data, and compliance needs — *what* is required, not specific package versions. The integration *design* (clients, pooling, failover) lives in the design doc.

## 10. Open Questions & Unresolved Decisions

What the grill did not settle. Each tagged with its grill node id (e.g. `C2`, `F3`). Never answered here — these are the calls still owed. If this section is large, note near the top that the spec is **partial**.

## 11. Out of Scope

Explicit non-goals, including concerns the grill deliberately dropped.

## 12. Validation Criteria

How to verify a built solution meets this spec's *what* — the product-level, behavioral checks. Technical verification (test seams, load/chaos, readiness) lives in the design doc.

## 13. Related / Further Reading

Paths to the grill run artifacts (initial-agenda / living-agenda / conversation-path), the companion `/to-design` doc once produced, and any other relevant docs.
```
</spec-template>
