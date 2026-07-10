---
name: survey-terrain
description: Survey the existing product, codebase, and organization that future work lands in, and compile a durable terrain pack — a lightweight map/index/glossary (glossary, product surface, system map, conventions & paved roads, constraints & gates, internal prior art) written to <repo>/.grill/terrain.md. The grill chain consumes the pack presence-based, so brownfield and enterprise runs start oriented while greenfield runs are untouched. Run once per repo; refresh as the terrain moves. Research only — it maps what exists; it decides nothing.
argument-hint: "(optional) repos, paths, or products to survey; say 'from scratch' to force a full resurvey"
disable-model-invocation: true
---

# Survey-Terrain

Brownfield work — a feature landing in an existing product, a system landing in an existing org — starts from a world that already exists, and the grill chain otherwise re-discovers that world from scratch: per lane, per run, per scout dispatch, with the findings evaporating into agenda items. This skill surveys the terrain once and compiles one durable **terrain pack**: a lightweight map, index, and glossary of the product surface, the systems, the conventions, and the org's gates. The chain consumes it **presence-based** — when a run holds a pack path, every planner, scout, facilitator, and tracer dispatch starts oriented instead of cold; when no pack exists, nothing anywhere changes. There is no brownfield mode: the pack's presence is the whole signal.

**You run the Scout and compile the pack. You do not interview, grill, or decide anything.** This is a non-interactive research pass with one artifact; there is no living agenda, no turn log, no lane dir, and no verdict to put to the user. The pack is a *map at map altitude*: systems and their responsibilities, owners, entry points, the org's real vocabulary — never file trees, never exhaustive indexes, never an attempt at ground truth.

## Doctrine — the three rules every consumer honors

These are stated once, here; the chain's skills carry only seam lines that lean on them.

- **The pack orients; it never resolves.** A pack fact is a scout-finding-class world fact: it seeds questions, sharpens recommendations, and supplies candidates — but it never closes a grill node `verified` or `mechanism` by itself. The chain's existing evidence discipline (a recorded instance, or a live scout verification) still applies to every load-bearing close. This is also the staleness guard: a pack that has drifted from reality can misdirect a question but never silently settle one.
- **Pack facts are never givens.** Givens are the *user's* stated convictions; pack content is *discovered* world-fact. An org gate or constraint the pack records surfaces as an agenda or constraint **candidate** the user confirms — it is never auto-filed into a lane's `givens.md`.
- **Staleness is stated, not feared.** The frontmatter's date and per-repo SHAs are the freshness signal. A consumer finding the pack behind the repo says so in one line — "surveyed at `<sha>`, repo has moved — orientation only" — and proceeds; orients-never-resolves makes a slightly stale pack safe. Scouts that find reality moved past the pack report the correction in their brief.

## Location, discovery & lifecycle

**The pack lives in the repo, gitignored:** `<repo>/.grill/terrain.md`. Writing it, ensure `.grill/` is ignored — check `git check-ignore .grill` and append `.grill/` to the repo's `.gitignore` if it isn't. One pack per repo the skill is run from; for work spanning repos, run from the primary repo and name the others as arguments — the survey's `gh` reach covers them in content, and there is no multi-pack registry to maintain.

**No repo, no pack.** Invoked outside any git repo, decline and say to run it from the repo the work lands in. A purely greenfield idea has no terrain to survey.

**Refresh, not patch.** Invoked where a pack already exists, enter **refresh mode**: read the existing pack, compute what moved since its recorded SHAs (`git log`/`git diff --stat` for the local repo; best-effort via `gh` for others), and dispatch each scout with the old pack as prior and the delta as focus — *verify and update what moved; don't re-derive the rest*. Then **rewrite the whole file** with fresh SHAs and date, written as current state — no per-section patching, no provenance annotations, no changelog inside the pack. Recommend a from-scratch resurvey instead — and run one when the user asks for it in the args — when the delta is structural rather than incremental: the pack is ancient, the org changed shape, or a grill run surfaced a pile of corrections.

**Generated-only.** The pack is a generated artifact and hand-edits are lost on refresh by design. Convictions the user wants held belong in a grill run as givens, or in real repo docs — never hand-maintained here, or every refresh becomes a merge.

**Never rewritten mid-run.** A grill run in flight keeps the pack it started with; ground doesn't shift under an in-flight interview. Refresh between runs.

## How to run this

- **One subagent, by name:** `grill-scout`, in its **grounding-survey** mode — the same worker the grills' Planner uses. Use the Agent tool (`subagent_type`); if `/agents` shows it plugin-scoped, use that form. No Planner, Facilitator, or Tracer here.
- **Scale the fan-out to the terrain.** A small solo repo gets **one** scout. A real product + org sweep gets **2–3 in parallel, split by surface**: the *product surface* (what the product does today, for whom, at what scale), the *system map* (repos/services, data stores, deploy topology, integration points, owners), and *conventions & gates* (paved roads, standards, compliance and review processes). Don't manufacture research the terrain doesn't warrant.
- **Dispatch concretely.** Each scout gets the scope to look in (which repos/paths/products, plus `gh` for the org-wide view), what its surface must come back with (mirroring the template sections it feeds), and — refresh mode — the existing pack plus the delta to focus on. The scout returns a sourced brief; it can't see this conversation.
- **Compile, don't paste.** Fold the briefs into the template below. Every line earns its place by being something a future grill question could lean on; where a section has nothing real, write `None found.` rather than padding. Facts the scouts couldn't confirm stay out or carry an explicit `(unconfirmed)`.
- **The file is the deliverable.** Write `<repo>/.grill/terrain.md` (overwriting on refresh, written as current state — no change narration), ensure the gitignore, then return only the absolute path plus a 1–2 line summary: scope surveyed, # repos/systems mapped, and anything major the survey couldn't confirm. Never paste the body into chat.

## The artifact — `terrain.md`

<terrain-template>
```markdown
---
title: Terrain — <product / org, one line>
date: <YYYY-MM-DD>
scope: <what was surveyed, one line>
repos:
  - <org/repo or path> @ <short sha>
---

## 1. Glossary

The org's and product's real terms — the vocabulary agenda items and specs should speak. One
line per term.

## 2. Product Surface

What the product does today: the major features and flows, the actors/roles as they exist now,
the promises users already rely on, and the rough scale facts (user counts, data volumes,
traffic) that product decisions and NFR targets hinge on.

## 3. System Map

The systems the work could land in or touch — coarse map altitude, entry points not file trees.
Per system: responsibility, **owner** (team / approver / on-call), key entry points. Then the
data stores and their ownership, the deploy topology, and the integration points between
systems and with the outside.

## 4. Conventions & Paved Roads

How things are done here: the standards, shared platforms, and blessed patterns a design is
expected to use — and where the org tolerates deviation.

## 5. Constraints & Gates

The org's non-negotiables and processes: compliance obligations, security/review/approval
gates and who grants them, platform mandates. These surface in grills as constraint
*candidates* for the user to confirm — never as auto-filed givens.

## 6. Internal Prior Art & Primitives

Adjacent internal tools, services, and primitives that bear on likely work: what each does,
where it lives, and what it could be adopted or composed for. Seeds opp-grill's adopt/buy
candidates and prior-art-check's compose-internal-primitives floor candidate.

## 7. Related

Load-bearing sources the survey leaned on: docs, dashboards, org pages, repos read via `gh`.
```
</terrain-template>

## How the chain consumes it

The consumption contract, so the chain's skills stay thin:

- **A pack path enters a run** by argument, by the user naming it, by chain-artifact frontmatter (`terrain:` in `idea.md` / `spec.md`), or by a silent file-exists check of `<repo>/.grill/terrain.md` at intake when the work lives in that repo. No other discovery.
- **In a run it lives in the Position block** as `terrain: <path or —>` and rides **every subagent dispatch** ("…and the terrain pack path when the run has one"), so planners, scouts, facilitators, and tracers read it directly.
- **The offer is one sentence, once.** An intake that finds no pack offers this skill in a single non-blocking sentence *only when the idea actually references an existing system* — the user names repos, paths, products, or a change to a live thing. Being invoked from inside a git checkout is never, by itself, the trigger.
- **Drift is one line.** An intake that finds a stale pack states the drift and moves on; a heavily drifted pack earns a refresh offer in the same sentence.
- **Corrections flow back through completion.** Scout-reported pack corrections land in the run's ordinary `meta` rows; a grill's completion summary mentions them (when any) and suggests a refresh.
