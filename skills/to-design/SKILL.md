---
name: to-design
description: Turn a completed grillmaster session + its to-spec PRD into a single, complete technical design document in the system temp dir — the *how* that realizes the spec's *what*: architecture, data/state, interfaces, execution, cross-cutting concerns, deployment/rollout, technical decisions (ADRs), risks, verification, and readiness. Invariants first; synthesis only — no re-interview, no invented architecture.
argument-hint: "(optional) path to a spec file or a grillmaster run dir; blank uses the most recent spec + its grill run"
disable-model-invocation: true
---

# to-design

Turn what a grilling settled — plus the product PRD that `to-spec` produced from it — into one durable **technical design document**: the *how* that realizes the spec's *what*. Where `to-spec` says what the system does and why, `to-design` says how it is built, run, and shipped so that it honors the spec.

This is the second conversion step in the chain: **grillmaster** (grill → understanding) → **to-spec** (product PRD) → **to-design** (technical design doc). It is a companion to the spec, not a replacement — it references the spec's ids, never restates its promises.

**Synthesize, do not interview.** The grill did the asking and the spec captured the *what*. Build the design from what was actually decided — never re-open the interview, and never invent architecture the grill didn't reach. A concern the grill left unexplored becomes an Open Question, not a guess.

The output is a single markdown file in the system temp dir, and it **leads with the invariants**.

**You are the orchestrator and the sole author.** You write the design in the main thread, then harden it with a small panel of read-only critics that edit nothing: an ad-hoc **Scout** for facts, and two opposing critics — an **Adversary** (*is it enough?*) and an **Auditor** (*is it faithful, and no more?*) — run over the draft in a two-round loop. The design doc, the spec, and the grill run are the shared state; there is no separate run directory.

## The boundary: what belongs here vs in the spec

`to-design` owns the *how*; `to-spec` owns the *what*. The test for any piece of content: **would it change if we re-implemented the same product a different way?** Yes → it's design (here). No → it's spec (leave it there, reference it).

- **Invariants** are carried forward by id — an `INV-` lives in the spec (the promise); §0 here re-states it as the constraint that binds the architecture, referenced by id. Never renumber or recreate spec ids.
- **NFR targets vs mechanisms** — the spec states the target (`p99 < 200ms`, `99.9%`); the design owns the mechanism that meets it.
- **Interface shape** — default a wire shape to the design; it belongs in the spec only when it's a frozen contract external consumers integrate against ("could a different implementation return a different shape and still honor this?" — yes → design).

## How to run this

- **Invariants first, literally.** Write the `## 0. Design Invariants & Constraints` block before any other section. Reasoning the binding constraints first is what keeps every component, contract, and decision below them consistent. Don't start §1 until §0 exists.
- **Required spine vs conditional sections.** Some sections are **cross-cutting and always present** — the frame every design doc needs, marked `[required]` below. The rest are **conditional**: include them only when the spec's scope and domain call for them, using each section's *include-when* trigger. You must **consciously decide each conditional section in or out** — "optional" means *deliberately judged*, never *silently forgotten*; the author self-check (step 5) is where you confirm each call. A stateless CLI has no deployment section; a non-AI tool has no AI section; a pure library has no operations section — omit what the domain doesn't have.
- **Posture dials depth, not the spine.** Read the run's `posture` (from the grill's Position block or the spec frontmatter — never re-derive it). Posture sets how hard each included section is pressed and the bar for "resolved enough": a `poc` design resolves the core path and honestly marks the rest; a `new-system` design isn't done until security, tenancy, data-lifecycle, DR, SLOs, capacity/cost, and dependencies are all pressed. Posture never removes a required section.
- **Never invent.** Every component, contract, `DD-`, and `RISK-` traces to something the grill settled, the spec states, or the user said — else it's tagged `(assumed — confirm)`. An included section with nothing settled uses the empty-state vocabulary below; it is never filled with best-practice boilerplate.
- **Four-state empty vocabulary.** A section (or subsection) with no settled content says exactly one of:
  - `N/A — <reason>` — the concern categorically doesn't apply to this system.
  - `None settled.` — it applies and matters, but the grill/spec never decided it, and it isn't blocking at this posture.
  - `→ OQ-###` — it applies, is unresolved, and is load-bearing enough that the design isn't buildable without it (route to §18).
  - `(assumed — confirm)` — an inline inference you had to make for coherence.

  Use `See §N` to cross-reference rather than duplicate. **Completeness is coverage of the *applicable* sections at the posture bar — never the absence of empty cells.**
- **No volatile detail.** Describe structure and flow at the design level — components, responsibilities, sequencing, schemas, rollout strategy, limits. Never file paths, class/function names, package versions, or code. The one exception: a schema or state-machine snippet that encodes a decision more precisely than prose.
- **No change narration.** The design describes the system as it is to be built, for a reader who never saw the grill. On a re-run, overwrite the file so it reads as written fresh.
- **The file is the deliverable.** Write it, then return only the absolute path + a 1–2 line summary (topic, posture, # decisions, # risks, # open questions, and **partial** if material areas are unresolved). Never paste the body into chat.
- **Dispatch the hardening panel by name.** The plugin ships four workers: `to-design-scout` (ad-hoc, to ground a fact), `to-design-auditor` and `to-design-adversary` (the two-critic hardening loop), and `to-design-alternatives` (only when a critic flags a hand-waved ADR). Dispatch them with the Agent tool (`subagent_type`); if `/agents` shows them plugin-scoped, use that form. You write the document; they only advise.
- **Critics are read-only; you are the pen — and there is no run directory.** Every critic returns a brief and edits nothing; you fold its findings in. The shared state is the **in-progress design doc, the spec, and the grill run** — no `.runs/` scaffolding. Every dispatch prompt carries the **dispatch contract**: the design-doc path, the spec path, the grill run dir, the **§0 Locked Invariants inline verbatim** (paraphrase is where intent inverts), the posture, and the critic's one job.
- **Subtract by default; restraint is first-class.** The two critics are opposing forces — the Adversary asks *is it enough?* and pushes complexity up; the Auditor asks *is it faithful, and no more?* and pushes it down. "Always add, never subtract" is the design's natural drift, so restraint is a co-equal mandate, not a footnote: every mechanism must earn its place against the posture, and an addition with no posture-justified need is cut, not kept.
- **Essentially non-interactive.** This is a conversion step, not an interview — there is one user gate, at the end. Unresolved load-bearing calls are reported as **partial** / **build-blocked**, never re-interviewed (that is Grillmaster's job). The one exception: if hardening shows the *spec* is wrong, stop and surface it rather than patch around it.
- **Posture-gated fan-out is a future extension.** At `new-system` the Adversary could split into parallel security / reliability / data-migration specialists (as the code-review panel does); today it runs as one whole-doc critic. Do not fan it out now.

## Find the inputs

`to-design` reads **two** inputs: the finished spec and the grill run behind it.

```bash
ls -dt "${TMPDIR:-/tmp}"/spec-*.md 2>/dev/null         # the to-spec PRDs
ls -dt "${TMPDIR:-/tmp}"/grillmaster-*/ 2>/dev/null     # the grill runs
```

- If the argument names a spec file or a grill run dir, use it. Otherwise take the most recent spec and its matching grill run (same `<slug>`).
- Read the spec in full — it is your source for the invariants, requirements, acceptance criteria, and product decisions you must honor and reference by id.
- Read the grill run's three files — `initial-agenda.md`, `living-agenda.md` (statuses + the `posture`), `conversation-path.md` (the decision trail) — for the technical decisions the grill settled and the concerns it left open.
- Also draw on this conversation. **Standalone fallback:** if no spec exists, synthesize from the grill run (or the conversation) alone and label the document **partial — no spec**; do not fabricate the spec's promises.

## Process

Steps 1–5 are yours alone — you locate, frame, and draft. Steps 6–7 add the read-only critics that harden what you wrote.

1. **Locate & read** the spec + grill run (above), or fall back to the conversation.
2. **Read the posture** and write **§0 first** — carry every spec `INV-`/`CON-` forward by id, add the technical non-negotiables the grill settled, and map each to where it's enforced (a section/decision) or to an Open Question. These are the **Locked Design Invariants** that travel verbatim to every critic.
3. **Decide the section set.** Keep every `[required]` section; for each `[conditional]` one, decide in or out from the spec's scope and domain using its include-when trigger.
4. **Route the material:** spec `REQ-`/`AC-` → the section that realizes them (trace by id in §21); grill `[resolved]` technical decisions → the body + §15 ADRs; grill `[unvisited]`/`[paused]` technical items → §18; grill `[dropped]` → §20; technical decisions moved out of the spec → §15.
5. **Draft the sections in order**, each grounded and consistent with §0, at the posture's depth, using the empty-state vocabulary where nothing is settled. Dispatch `to-design-scout` **ad-hoc** whenever a load-bearing fact would otherwise be a guess, and fold its finding in with `(verified: <url-or-path>)`. Then run the cheap **author self-check**: re-read §0, confirm each conditional section's in/out call, and run the 7-slice coverage self-check (external contracts / AI / data & state / internal architecture / failure modes & ops / verification — *behavior* is the spec's territory) — catch the obvious before spending critics.
6. **Hardening round 1.** Dispatch `to-design-auditor` and `to-design-adversary` **in parallel** (one message), each with the dispatch contract. If either flags a hand-waved ADR, dispatch `to-design-alternatives` for that one `DD-`. Then revise the doc, triaging every finding by the rules below.
7. **Hardening round 2.** Re-dispatch the same two critics on the **revised** doc — the independent check must see the hardened version, not the first draft — and revise again. Two rounds total. A finding that survives round 2 as VIOLATED or blocking is resolved or explicitly routed to §18/§16 before you finish.
8. **Final pass.** Fill §21 traceability; delete or convert any design element that traces to nothing; confirm every component/contract/`DD-` still honors §0.
9. **Save & report.** Write to `${TMPDIR:-/tmp}/design-<slug>.md` (same `<slug>` as the spec). Return the absolute path + the 1–2 line summary; say **partial** / **build-blocked** if load-bearing questions remain.

**Triaging a finding** (steps 6–7) — this is what keeps *hardened* compatible with *never invent*:

- **Under-hardened, and the grill/spec gave enough** → harden it to the posture bar.
- **Over-built for the posture** (an Auditor restraint finding) → **subtract**: cut the gold-plating, the speculative generality, the one-caller abstraction. Simplicity wins ties.
- **Adversary says "add" vs. Auditor says "cut"** → adjudicate against the **posture**: an addition earns its place only if the posture justifies the risk it closes; otherwise the default is *not* to add. `poc` leans hard toward simple; `new-system` tolerates more machinery.
- **Invented — traces to nothing** → delete it, or downgrade to `(assumed — confirm)`.
- **Load-bearing but genuinely unreached by the grill** → route to `§18 OQ-` / `§16 RISK-`; never guess.
- **The finding reveals the *spec* is wrong** → stop and surface it to the user ("this can't be hardened without reopening REQ-###"); never silently patch around a bad input.

## Design-doc template

Write the file in this shape. `[required]` sections always appear; `[conditional]` sections appear only when their include-when trigger fits the spec. Coded ids (`DC-`, `DD-`, `RISK-`, `OQ-`) are sequential within prefix; spec ids (`INV-`/`REQ-`/`AC-`) are referenced, never recreated.

<design-template>
```markdown
---
title: <what's being built — technical>
status: draft
date_created: <YYYY-MM-DD>
source: <grill run dir, or "conversation">
spec_source: <spec file path, or "none — partial">
posture: <poc | internal-tool | product-feature | new-system>
tags: [<relevant tags>]
---

## 0. Design Invariants & Constraints  [required]

The constraints the design must honor, reasoned first. Carry spec invariants forward by id; add technical non-negotiables the grill settled. Each maps to where it's enforced or to an Open Question.

- **INV-001** (from spec): <one-line non-negotiable> — enforced by: <§/DD, or → OQ-###>
- **DC-001**: <technical constraint the grill settled> — enforced by: <…>

## 1. Design Overview  [required]

A 3–5 sentence tl;dr of the technical approach and the shape of the solution. Links the spec it realizes.

## 2. Goals & Non-Goals (technical)  [required]

Design goals (e.g. minimize new infra; sub-second p99) and explicit technical non-goals. Distinct from product scope.

## 3. Context & Existing System  [required]

What already exists and this builds on (from grill grounding), the environment it slots into, assumptions, and soft/environmental constraints (hard ones are in §0).

## 4. Dependencies & External Systems  [conditional — include when the system relies on external services, libraries, or platforms]

Each dependency: what it's used for, owner, SLA/quota, failure behavior, and version/supply-chain risk.

## 5. System Architecture  [required]

System context (users + external systems) and the component decomposition: each component's responsibility, the boundaries between them, and the dependency direction. A decision-encoding diagram if it clarifies. **Tenancy/isolation model:** <single | shared | isolated | N/A> (one line; cross-ref §6/§12). *Static structure — what exists.*

## 6. Domain Model, Data & State  [conditional — include when the system owns persistent data, entities, or non-trivial state]

Entities, relationships, aggregates; state machines / lifecycle / transitions; derived state; data ownership; persistence design (storage choice, design-level schema, access patterns, consistency model).

- **Data retention, deletion & compliance:** TTL/retention, soft/hard/cascade delete, right-to-erasure propagation across derived stores/caches/backups/logs, PII classification. (`N/A` only if no persisted or user data.)
- **Migration & backfill:** schema/data migration approach and sequencing.

## 7. Interfaces & Contracts  [conditional — include when the system exposes an API, CLI, events, or a library surface]

Internal component contracts + the *realization* of external surfaces: schemas, field types, status/error codes, idempotency keys, pagination.

- **Versioning & deprecation policy:** how the contract evolves; N-1 compatibility. (`N/A` if no durable public contract.)
- **i18n / a11y:** localization and accessibility. (Include only for user-facing UI surfaces; else `N/A`.)

## 8. Execution & Data Flow  [conditional — include when there is non-trivial runtime flow, orchestration, or async work]

Key operation/request lifecycles, sequencing, sync vs async, orchestration, background jobs/queues/schedulers; how data flows across components.

- **Concurrency, ordering & caching:** parallelism, locking, transactions, race conditions, delivery guarantees, cache strategy & invalidation.

*Dynamic behavior — references the components (§5), entities (§6), and contracts (§7).*

## 9. AI / LLM Design  [conditional — include when the system has AI/LLM behavior]

Model/agent responsibilities; the human-vs-AI boundary; prompt/context/retrieval/memory/tool design; how nondeterminism is handled; quality/eval strategy; guardrails; fallback / human review; cost & latency budgets (cost → §10, eval → §17).

## 10. Capacity, Performance & Cost  [conditional — include when performance, scale, or cost is material (product-feature/new-system, or notable AI/cloud cost)]

Expected load (QPS, data growth), the capacity envelope, latency/throughput budgets and the mechanisms that meet the spec's NFR targets, bottlenecks, scaling strategy, and a rough cost model (incl. LLM token-cost × volume).

## 11. Failure Modes, Reliability & Recovery  [conditional — include when the system runs as a service or performs failure-prone operations]

Failure modes & edge cases; retries/timeouts/backoff; circuit breakers; graceful degradation; partial failure & blast radius.

- **Backup & disaster recovery:** RTO/RPO, backup/restore approach. (`N/A` for stateless/ephemeral.)

## 12. Security & Privacy  [conditional — include when the system handles users, data, auth, or untrusted input]

Trust boundaries; authn/authz model; threat model (key threats + mitigations); input validation; secrets management; data protection/encryption; abuse/rate-limiting; privacy & compliance obligations.

## 13. Observability & Operability  [conditional — include when the system is deployed and operated]

- **SLIs / SLOs / error budgets** — what defines success and the budget that governs rollout risk (alerting derives from these).
- Logging, metrics, tracing, alerting; health signals.
- **Product analytics / instrumentation:** the events that measure adoption/funnel and their schema/emission points (distinct from ops telemetry).
- Config & feature-flag surface; runbooks / incident response.

## 14. Deployment, Rollout & Rollback  [conditional — include when the system is deployed]

Deployment architecture (environments, topology, runtime — design level); rollout strategy (feature flags, canary/blue-green, migration sequencing); rollback/reversibility; backward compatibility across the rollout; **feature-flag lifecycle** (owner / rollout condition / expiry); **config & secrets management**; launch/readiness checklist.

## 15. Technical Decisions & Alternatives (ADRs)  [required]

The significant technical decisions, each referenced inline where the design first relies on it.

- **DD-001**: <decision> — options weighed: <…>; chosen because <…>; rejected: <…>. (honors INV-001; from grill turn/node or spec)

## 16. Technical Risks & Mitigations  [required]

- **RISK-001**: <what could go wrong> — impact: <…>; mitigation/contingency: <…>.

## 17. Test & Verification Strategy  [required]

What makes a good test here (assert external behavior, not implementation). The seam(s) to test at — prefer existing and the highest/fewest, ideally one. Test levels (unit/integration/e2e/load/chaos), fixtures, environments. Traces spec `AC-` ids.

## 18. Open Questions & Unresolved Technical Decisions  [required]

The technical calls the grill/spec didn't settle. Each tagged with its grill node/turn or spec id. Never answered here.

- **OQ-001**: <question> — <source: grill node / spec id>

## 19. Implementation Plan & Readiness  [required]

Build phasing/sequencing (dependency-ordered) and the readiness criteria that make this design safe to build. **Dev experience / local-run:** how a developer runs, seeds, mocks, and checks it locally.

## 20. Out of Scope (technical)  [required]

Technical non-goals and deferred design, including grill `[dropped]` technical items.

## 21. Traceability  [required]

Every spec id maps to the design that realizes it; a design element tracing to nothing is fabrication (delete) or a new decision (make it a sourced `DD-`).

| Spec id | Realized by |
| --- | --- |
| INV-001 | §0, DD-003 |
| REQ-004 | §6, §7 |
| AC-002 | §17 |

## 22. Related / Further Reading  [required]

The spec file, the grill run artifacts (initial-agenda / living-agenda / conversation-path), and any prior-art docs.
```
</design-template>
