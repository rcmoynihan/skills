---
name: to-spec
description: Formalize a completed grillmaster session (or the current conversation) into a single PRD/spec markdown file in the system temp dir — invariants/non-negotiables enumerated first, then a blend of product framing (problem, solution, user stories) and a rigorous, testable spec (coded requirements, interfaces, acceptance criteria, validation). Synthesis only — no re-interview.
argument-hint: "(optional) path to a grillmaster run dir; blank uses the most recent run + this conversation"
disable-model-invocation: true
---

# to-spec

Turn everything a grilling settled — the decisions, clarifications, and constraints — into one durable spec. This is the conversion step Grillmaster's completion offers: the grill produced *understanding*; this produces the *document*.

**Synthesize, do not interview.** The grill already did the asking. Build the spec from what was actually decided and from the conversation — never re-open the interview, and never invent an answer the grill didn't reach. Where the grill left a concern unexplored, the spec says so; it does not fill the hole with a guess.

The output is a single markdown file written to the system temp dir, blending product framing (problem, solution, user stories) with a rigorous, machine-readable spec (coded requirements, interfaces, acceptance criteria, validation) — and it **leads with the invariants**.

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

Explicit, testable statements. A constraint that restates an invariant references its id.

- **REQ-001**: <requirement>
- **CON-001**: <constraint>  (e.g. "honors INV-001")
- **GUD-001**: <guideline / recommendation>

## 6. Interfaces & Data Contracts

APIs, CLI surfaces, events, data shapes, schemas — the *what*, not the *how*. Tables/code blocks for schemas.

## 7. Acceptance Criteria

Testable, in Given-When-Then form, tied back to requirement ids.

- **AC-001**: Given <context>, when <action>, then <expected outcome>.  (REQ-001)

## 8. Testing Strategy

What makes a good test here (assert external behavior, not implementation). The seam(s) to test at — prefer existing and the highest/fewest, ideally one. Test levels and any prior art to mirror.

## 9. Key Decisions & Rationale

The decisions and clarifications the grill produced and *why* — architectural calls, schema/API choices, interaction models. Decision-encoding snippets only.

## 10. Dependencies & External Integrations

External systems, services, platforms, data, and compliance needs. State *what* is required, not specific package versions.

## 11. Open Questions & Unresolved Decisions

What the grill did not settle. Each tagged with its grill node id (e.g. `C2`, `F3`). Never answered here — these are the calls still owed. If this section is large, note near the top that the spec is **partial**.

## 12. Out of Scope

Explicit non-goals, including concerns the grill deliberately dropped.

## 13. Validation Criteria

How to verify a built solution complies with this spec.

## 14. Related / Further Reading

Paths to the grill run artifacts (initial-agenda / living-agenda / conversation-path) and any other relevant docs.
```
</spec-template>
