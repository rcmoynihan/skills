# Global rules

These apply to every project on this machine, in any tool.

## Be useful, not agreeable

Your job is to be helpful, not to agree. If the user is heading somewhere wrong, say so — politely,
firmly, and with evidence. Prefer precise, constructive pushback over hedging or flattery, and lead
with a recommendation rather than an exhaustive survey of options. Standing your ground on a correct
point serves the user better than reinforcing a mistaken one.

## Scope and change discipline

Make the smallest change that fully solves the stated problem — grounded in what the code and the
user actually say, verified before you stop. **Do less, ask more.**

- **Only what was asked.** Touch only what the task requires. No drive-by reformatting, renaming,
  refactoring, or generalizing beyond the request. Spot an adjacent improvement? Note it and ask —
  don't fold it in.
- **As simple as the problem.** Start simple and apply YAGNI: no abstraction, layer, config, or
  defensive machinery until the concrete case needs it. Solve today's single case, and match the
  altitude of the code around you rather than introducing a second way to do something that already
  has one.
- **Ground before acting.** Read the whole path and all callers before editing; confirm every file,
  API, and signature you reference actually exists rather than assuming it. Resolve ambiguity by
  asking, not by writing code for every interpretation.
- **Solve the real problem.** Reproduce the reported failure before fixing it, address the root
  cause rather than the symptom, and honor stated constraints (dependencies, style, compatibility,
  "don't touch X").
- **Verify, then stop.** Run it — build, tests, the original repro — before saying "done" or
  "fixed." Re-read the full request and confirm every requirement is met, not just the easy ones.

The full catalog of these failure modes, with examples per pitfall, lives in the
`coding-agent-pitfalls` skill — consult it when reviewing code (yours or another agent's) for
anti-patterns.

## No legacy or dead code

- Treat changes as hard cutovers, not migrations, unless explicitly told otherwise. Call them
  "implementation plans," not "migration plans."
- When you change or supersede code, delete the old path — do not retain it behind
  `legacy`/`deprecated`/`TODO remove` comments or `_v2`-style suffixes. Keep one authoritative
  implementation, actively used and tested.
- Never write a test that merely asserts the codebase moved from state A to state B. The code should
  not reference past flows, schemas, or names.
- Historical context belongs in version control, not in dead code.

## Prefer one canonical path

Design each piece of code around a single canonical happy path, and make the inputs and contract
guarantee that path is valid — rather than baking in extra branches to tolerate a messy contract.
When a value can arrive in several shapes, normalize or reject it at the boundary so everything
downstream sees one shape; don't thread `if`/`else` variants through the core logic. When you feel
the urge to add a branch, first ask whether the contract should have made that case impossible.

Avoid, unless a concrete requirement demands it:
- **Fallbacks** that silently substitute a default or alternate route when the primary one fails —
  they hide errors and turn one predictable path into several. Prefer to fail loudly, or fix the
  cause so the primary path holds.
- **Dual-version or compatibility paths** kept beside the real one "just in case" (see *No legacy or
  dead code*) — one authoritative implementation, not old-way/new-way branches.
- **Speculative defensive code** — try/except, null-guards, or validation for conditions the caller
  or contract has already ruled out (see *Scope and change discipline*). Guard real, reachable
  failure modes, not imagined ones.

A second path is justified only when the requirement genuinely has two cases (a real feature branch),
not when it's compensating for uncertainty, legacy shapes, or unowned inputs. There, resolve the
uncertainty or tighten the contract instead.

## No change narration

Every artifact you produce must describe the system **as it currently is**, written for a reader who
neither knows nor cares that a change just happened. **Never narrate the change itself** unless the
user explicitly asks, or you are writing a literal changelog.

"Change narration" is any of:
- what changed, or what it used to be (before-vs-after);
- why *you made the change*, or the steps/process you took to make it;
- your own **newly-acquired understanding** of code that already existed — a realization you had
  while working is not documentation.

This is banned in **source code, inline comments, docstrings, Markdown/READMEs/docs, and PR
bodies/descriptions**. The **only** exception is a literal changelog file (e.g. `CHANGELOG.md`),
where describing the delta is the document's purpose. (Commit messages describing the diff are also
fine — that is what they are for.)

**Comments used in moderation; code should self-document.** Default to clear names, small functions, and
obvious structure, and add **no comment** when the code already reads clearly. The one kind of
comment that is *not* change narration — present-tense rationale for *why the code is the way it is*
— is still only warranted when **both** hold: (1) the code cannot reasonably be made
self-explanatory on its own, and (2) an average developer would not reasonably know or remember that
reason on a first read. This is a narrow exception, not a license to annotate; when in doubt, leave
it out. Even a warranted comment must still pass the change-narration test: *would it make sense to
someone reading a fresh checkout who never saw the prior version?* If it only makes sense as "this
is different / here's what I did / here's what I figured out," delete it.

**Discovering existing behavior is not a license to document it.** If, while working, you figure out
how some pre-existing code behaves, do **not** automatically add or edit comments/docstrings/docs to
record that. If you believe it genuinely warrants documentation, propose it and get approval first.

### Examples

(The ✅ code comments below show the *right form* for the rare case where a comment has already
cleared the bar above — they are not a suggestion to add comments.)

✅ Present-tense rationale (no reference to the change):
```python
# Called directly so the refusal is served even when the agent wouldn't select the tool.
refusal = await prepared_responses_tool.run(question)

def serve_refusal(query: str) -> str:
    """Return the prepared refusal for an out-of-scope query."""
```

❌ Narrates the change / prior state / your process:
```python
# Changed from a restricted agent to a direct call because the agent could skip the tool.
# Previously this went through retrieve_graph; we now short-circuit here.
refusal = await prepared_responses_tool.run(question)

def serve_refusal(query: str) -> str:
    """Added in CB-1234 to fix out-of-scope leakage. This used to be handled by the agent."""
```

❌ Documenting your own newly-acquired understanding of existing behavior:
```python
# I traced this and confirmed parse_config() already validates the schema, so no need here.
data = parse_config(raw)
```
→ Correct action: add nothing. Your having just learned it is not a reason to annotate it.

✅ PR body / Markdown — describes the resulting behavior and rationale:
```text
## Out-of-scope handling
Out-of-scope requests are refused via a direct prepared-response call before any retriever
runs; the refusal text comes from the prepared-response store.
```

❌ PR body / Markdown — narrates the journey, before/after, and discoveries:
```text
## What I did
I first tried a restricted agent, but discovered it could skip the tool, so I switched to a
direct call and deleted the old fallback. I then realized the prefix broke the similarity
search, so I removed that too.
```

**Why:** code and docs are read far more often than changed, almost always by people without the
change in mind. Change narration rots immediately (the "new" way becomes the only way; the
"previous" behavior is gone from the reader's world), misleads readers into mistaking history for
current state, and bloats artifacts with process noise. Version control already records what changed
and why. Keeping every artifact a clean description of the present state is one of the
highest-leverage things you can do.

## Don't create markdown files to explain changes

- Do not create a standalone markdown file just to explain or narrate a change.
- Create markdown only for enduring documentation of core modules; docs for a one-off refactor, a
  test run, or profiling do not count.
- Prefer inline docstrings or an existing README over a new file.
