---
name: learn-from-pr-comments
description: Turn the issues a PR's reviewers raised into minimal, de-duplicated rule additions for the code-review reviewer profiles, so we catch the same class of issue ourselves before the next human review. Reads the response plan (and any resulting diff), keeps only comments that generalize, verifies existing coverage, and proposes profile edits — applying only what the human approves. Use after planning/resolving a PR's comments, to feed the lessons back into our own review.
argument-hint: "(optional) path to a pr-*-response-plan.md; defaults to the newest one in the plugin's temp dir"
disable-model-invocation: true
---

# Learn from PR comments

You close the feedback loop on our own review. A human reviewer caught something on this PR; your
job is to decide which of those catches generalize into a reusable rule, confirm we don't already
cover them, and fold the survivors into the existing `code-review` reviewer profiles — so the same
class of issue is caught by our review *before* the next PR reaches a human.

You **propose first and apply only what the human approves.** The one thing you edit is the
`code-review` reviewer profiles (`agents/code-review-*.md`); you never touch PR threads, commits,
or pushes.

Hold these five constraints above everything else — the whole method serves them:

- **Moderate generalizability bar.** Do not propose a rule per comment. Only patterns likely to
  recur across future PRs survive. Err toward dropping.
- **Never duplicate a rule.** Before proposing anything, be sure no existing rule already covers
  it. When one does, either strengthen its wording or record it as an already-covered recall miss —
  never add a near-duplicate bullet.
- **One profile per rule, never a new profile.** The panel of perspectives is assumed complete;
  every addition lands in the single best-fit existing profile.
- **No recency bias.** A new or amended rule must not outweigh, out-shout, or get hoisted above its
  siblings.
- **Always human-approved.** Present the proposal and wait; apply only the approved (possibly
  modified) subset.

**Honor the global rules:** no change-narration in the profiles you edit. An added or amended rule
must read as if it always existed — never "added after PR #…", "previously we missed…", or any
before/after framing. The profiles describe what the reviewer hunts for, in the present tense.

## Step 1 — Load the source lessons

Resolve the response-plan file: use the path argument if given, otherwise pick the newest match of
`"${TMPDIR:-/tmp}"/code-goblin-pro/pr-*-response-plan.md` (same discovery contract
`execute-pr-response-plan` uses):

```bash
ls -t "${TMPDIR:-/tmp}"/code-goblin-pro/pr-*-response-plan.md 2>/dev/null | head -1
```

Read it and extract the **PR ref** and, for every `C#` block, its **literal ask**, **spirit**,
**risk if ignored**, and **decision** (implement / alternate / push back / need info / defer). If no
plan file is found, stop and ask for one — this skill runs after at least `plan-pr-comment-responses`.

If the PR's branch is checked out or reachable, also read what the comments actually changed
(`gh pr diff <ref>`, or `git log`/`git diff` on the head branch). Seeing the concrete fix sharpens
the generalization and tells NEW from ALREADY-COVERED more reliably than the plan text alone.

## Step 2 — Select candidate lessons (generalizability filter)

A comment becomes a candidate only if **both** hold:

- **We agreed the reviewer had a point** — its decision was `implement` or `implement alternate`.
  Exclude `push back` (we disagreed, so there is no lesson to learn), and `need info` / `defer`
  (unresolved). A push-back earns reconsideration only if it exposes a real, recurring gap — e.g.
  reviewers repeatedly confused by the same unclear structure — which is the exception, not the rule.
- **It clears the moderate generalizability bar** — the issue is a pattern that would plausibly
  recur on future PRs across this codebase, not a one-off tied to this feature's domain logic, a
  subjective preference, a naming nit, or a fact true only of this file. State the candidate as a
  single *general* rule sentence; if you can't state it without naming this PR's specifics, it fails
  the bar. When unsure, drop it — the cost of a missed lesson is low; the cost of rule spam is high.

Record every comment you drop with a one-line reason. The dropped list stays visible in the
proposal so the human can override a call you made.

## Step 3 — Existing-coverage check (the anti-duplication gate)

For each surviving candidate, identify the one owning perspective (Step 4) and **read that profile**
— plus any sibling it might plausibly belong to — before proposing anything. Resolve to exactly one
of three outcomes:

- **NEW** — no existing bullet covers this pattern. Propose one new minimal bullet in the best-fit
  profile.
- **AMEND** — a bullet exists but is too vague or weak to have caught this, or is missing the
  telling concrete example that would have. Propose tightened wording or an added example on the
  *existing* bullet. Never add a second near-duplicate bullet beside it.
- **ALREADY COVERED (no change)** — an existing bullet covers it adequately; the review simply
  missed it this time. Propose **no** edit. Record it as a recall miss. If the likely cause is that
  `code-review-lead` didn't spawn the owning profile for this diff, note that as a selection-logic
  observation for the human — do not add or duplicate a rule to compensate for a spawn miss.

Bias toward AMEND and ALREADY-COVERED over NEW. Most real lessons are refinements of a rule we
already have, not gaps in the panel.

## Step 4 — Assign to one existing profile, never a new one

Map each NEW/AMEND rule to the single perspective that owns its dimension. The profiles live at
`agents/code-review-*.md`; their dimensions are:

- **correctness** — logic / behavioral bugs, boundaries, null propagation, races, state transitions.
- **testing** — coverage gaps, false-confidence or brittle tests, missing error-path tests.
- **maintainability** — structural complexity, duplication, magic values, type-model smells.
- **yagni** — overengineering, unused flexibility, needless defensive paths, missed reuse.
- **security** — injection, auth/authz bypass, secrets, deserialization, SSRF/path traversal.
- **performance** — N+1, unbounded memory, missing pagination, hot-path cost, blocking async I/O.
- **api-contract** — breaking interface/response-shape/type changes, versioning, error-shape drift.
- **data-migration** — schema/backfill safety, drift, migration reversibility and observability.
- **reliability** — error handling at I/O boundaries, retries/backoff, timeouts, cascading failure.
- **project-standards** — compliance with the repo's own `CLAUDE.md` / `AGENTS.md` written rules.
- **adversarial** — emergent, cross-component, composition, and abuse failures (the catch-all for
  patterns that span reviewers).
- **previous-comments** — verifying prior review feedback was actually addressed.
- **agent-native** — agent/tool-surface parity for user-facing actions.

Use each profile's own `## What you don't flag` hand-offs to break ties. If a candidate fits no
existing perspective, that is a signal it isn't a code-review rule at all — drop it, or surface it
to the human as out-of-scope. **Never create a new profile**, and do not touch the `code-review`
skill or `code-review-lead` unless an approved lesson genuinely needs a new conditional-spawn
trigger (rare — treat that as a separate, explicitly-flagged change, not a default).

## Step 5 — Draft each edit under the no-recency-bias discipline

Match the target file's exact format. A rule is a bullet under `## What you're hunting for` (at the
top level in flat profiles, or under the most relevant `### ` subsection in the richer ones):
`- **<bold lead phrase>** <sep> <example-laden sentence, often ending in a trace/verify
instruction>.` Match the file's separator — most profiles use a literal `--`; maintainability and
yagni use `—`. A non-finding goes under `## What you don't flag` instead.

Hold the drafting to these limits:

- **Proportion.** A new or amended bullet must not exceed the length or emphasis of the median
  sibling bullet in its section. No `IMPORTANT`, `ALWAYS`, all-caps, or extra bold beyond what
  siblings already use.
- **Placement.** Append in natural order at the end of the relevant list or subsection. Never hoist
  a rule to the top, and never restructure a section to foreground the new one.
- **Prefer AMEND.** Strengthening an existing bullet beats adding an adjacent near-neighbor.
- **Cap volume.** Aim for at most ~1–2 additions per profile per run. If more survive, keep the
  highest-value ones and list the rest as deferred for the human to decide, rather than bulk-adding.

## Step 6 — Adversarial pass via `codex:rescue` (optional)

Invoke the **`codex:rescue`** skill (via the Skill tool). Hand it your candidate list and the target
profiles, and ask it to **kill** each proposed rule: argue it is already covered, too narrow to
generalize, or would overshadow its siblings. Fold in convincing refutations — demote a rule to
ALREADY-COVERED or drop it. Keep a proposal where its critique isn't convincing; you have the final
say. If `codex:rescue` reports Codex is unavailable, note that and proceed.

## Step 7 — Write the proposal and present for approval

Write the proposal to `"${TMPDIR:-/tmp}/code-goblin-pro/pr-<ref>-review-lessons.md"` (derive `<ref>`
from the plan file's PR ref). For each proposed change include: the source `C#`(s) and their
decision → the generalized rule → why it clears the generalizability bar → the coverage verdict
(NEW / AMEND / ALREADY-COVERED) and which profile(s) you read to reach it → the target profile and
section → the exact bullet text to add, or the before/after for an amend → its placement. Add a
**Dropped candidates** section (comment → reason) and an **Already-covered / recall-miss** section.

```markdown
# Review lessons from PR <ref>

## Proposed changes
### L1 — <short title>
- Source: C3 (implement), C7 (implement alternate)
- Generalized rule: <one sentence>
- Why it generalizes: <recurrence rationale>
- Coverage verdict: NEW | AMEND | ALREADY-COVERED — profiles read: <names>
- Target: agents/code-review-<dimension>.md → ## What you're hunting for
- Exact text / before→after:
  <the bullet, or the amended bullet's before and after>
- Placement: append at end of section
(repeat per change)

## Dropped candidates
- C2 — naming nit, not generalizable
- C5 — domain-specific to this feature

## Already-covered / recall-miss
- C4 — covered by code-review-reliability "missing timeouts"; review missed it this run
  (lead may not have spawned reliability for this diff)
```

Then present the proposal in chat and **wait for explicit approval.** The user may approve all, deny
individual changes, or modify wording. This gate is unconditional — it fires on every run, including
when `auto-resolve-pr-comments` invokes you as its final step. Apply nothing before approval.

## Step 8 — Apply the approved subset

After approval, apply only the approved (and any user-modified) edits to the `agents/code-review-*.md`
files. Respect the no-change-narration rule from the intro. Then report to the user: which rules were
applied and where, which were denied or deferred, the already-covered/recall-miss notes, and the
proposal artifact path. If nothing was approved, apply nothing and say so.

## Quality bar

- Every surviving candidate resolves to exactly one verdict — NEW, AMEND, or ALREADY-COVERED — with
  the profile(s) you read named as evidence.
- No duplicate rules: an existing-coverage check precedes every proposed addition, and ALREADY-COVERED
  produces no edit.
- Every added/amended rule lands in exactly one existing profile, matches its sibling format and
  separator, stays within sibling length/emphasis, and is appended (never hoisted). No new profile.
- The dropped-candidates list is populated — a run that generalizes every comment has not applied the
  bar.
- Nothing is applied without explicit approval, and the skill edits only reviewer profiles — never PR
  threads, commits, or pushes.
- Re-running on the same plan after applying should resolve every applied lesson to ALREADY-COVERED
  and propose no further edits.
