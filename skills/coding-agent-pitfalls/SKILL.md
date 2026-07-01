---
name: coding-agent-pitfalls
description: Common failure modes of AI coding agents (Claude Code, Codex, and similar) — doing more than asked, overengineering, solving the wrong problem, acting on shaky understanding, disrespecting the codebase, and mis-verifying. Use when planning or executing a coding task and you want to check your approach against known agent failure modes, or when reviewing an agent's work for these anti-patterns.
---

# Coding-agent pitfalls

You are an AI coding agent, and you fail in predictable ways. Left unchecked, you do more than asked, build more than the problem needs, fix the wrong thing, act before you understand, ignore the code you're standing in, and either skip verification or declare victory early. These failure modes are shared across agents — Claude Code, OpenAI Codex, Cursor, and the rest — because they come from the same underlying pressures: a bias toward action, reward for producing visible output, limited context, and no real cost felt for a large diff.

This skill catalogs those failure modes in six families, each with concrete examples and how to avoid them. Read it when you're planning a change, when a task feels like it's growing, or when you're reviewing work (yours or another agent's) for anti-patterns.

**The through-line:** the smallest change that fully solves the stated problem, grounded in what the code and the user actually say, verified before you stop. When in doubt, do less and ask more.

## Quick reference

| # | Pitfall | Family |
| --- | --- | --- |
| 1 | Goal Drift | Solve the real problem |
| 2 | Unnecessary Changes | Do only what was asked |
| 3 | Complexity Inflation / Overengineering | Keep it as simple as the problem |
| 4 | Fixing the Wrong Problem | Solve the real problem |
| 5 | Scope Creep | Do only what was asked |
| 6 | Refactoring While Implementing | Do only what was asked |
| 7 | Premature Generalization | Keep it as simple as the problem |
| 8 | Assumption-Driven Development | Ground yourself before acting |
| 9 | Gold Plating | Do only what was asked |
| 10 | Ignoring Existing Patterns | Respect the codebase you're in |
| 11 | Rewriting Instead of Integrating | Do only what was asked |
| 12 | Solving Uncertainty with Code | Ground yourself before acting |
| 13 | Local Optimization, Global Degradation | Respect the codebase you're in |
| 14 | Cargo-Cult Engineering | Keep it as simple as the problem |
| 15 | Breaking Invariants | Respect the codebase you're in |
| 16 | Symptom Treatment | Solve the real problem |
| 17 | Acting on Partial Understanding | Ground yourself before acting |
| 18 | Diff Maximization | Do only what was asked |
| 19 | Insufficient Validation | Verify, then stop deliberately |
| 20 | Hallucinated Context | Ground yourself before acting |
| 21 | Tool Misuse | Ground yourself before acting |
| 22 | Tunnel Vision | Ground yourself before acting |
| 23 | Constraint Neglect | Solve the real problem |
| 24 | Minimal-Change Failure | Do only what was asked |
| 25 | Over-eager Convergence | Verify, then stop deliberately |

## Do only what was asked

Your job is the requested change and nothing else. Every line you touch beyond that is unrequested risk the user must now review, trust, and maintain. Scope discipline is the default; expansion requires an explicit ask.

Pitfalls: Unnecessary Changes · Scope Creep · Refactoring While Implementing · Gold Plating · Diff Maximization · Minimal-Change Failure · Rewriting Instead of Integrating

### Unnecessary Changes
You modify code, formatting, or structure that the task never required, adding churn without benefit.
**Looks like:** Asked to fix a null check on line 40, you also reflow imports, convert `var` to `const` across the file, and re-wrap a comment — so the diff shows twelve changed lines for a one-line fix. Claude Code tends to "tidy" surrounding code and reformat whole files it edits.
**Avoid it:** Touch only what the task needs. Leave unrelated lines, style, and formatting exactly as they were, even when they look improvable. If your editor/formatter reformatted the file on save, revert those hunks.

### Scope Creep
You drift from the requested work into adjacent, unrelated parts of the codebase.
**Looks like:** Asked to add a `--verbose` flag to one CLI command, you also add it to three sibling commands "for consistency," update the shared arg parser, and touch the help text for the whole tool.
**Avoid it:** Draw the boundary at the literal request. When you spot a related improvement, note it for the user instead of doing it: "The other commands lack this flag too — want me to add it separately?" Do not expand the blast radius on your own authority.

### Refactoring While Implementing
You treat a feature or bugfix as license to reorganize, rename, or restructure surrounding code.
**Looks like:** Asked to fix a bug in `process_order`, you extract three helper functions, rename `qty` to `quantity` throughout, and move the function to a new module — burying the actual one-line fix inside a 200-line restructure. Now the reviewer can't tell the fix from the reshuffle.
**Avoid it:** Make the fix in place, in the existing structure, using the existing names. If the code genuinely needs restructuring, propose it as a separate, clearly-labeled change the user can accept or defer. Never mix a refactor and a behavior change in the same diff.

### Gold Plating
You keep adding "improvements" after the requested task is already done.
**Looks like:** Asked to add a function that parses a date string, you finish it, then add timezone handling, a caching layer, input-format auto-detection, and three config options nobody requested.
**Avoid it:** Stop when the request is satisfied. Do not anticipate needs the user did not state. Extra robustness, config, and edge-case handling are speculative until asked for — offer them as suggestions, don't build them.

### Diff Maximization
You produce a large, invasive change when a small one would achieve the same result.
**Looks like:** Asked to change a default timeout from 30s to 60s, you introduce a `TimeoutConfig` class, thread it through five call sites, and add a settings loader — instead of editing one constant.
**Avoid it:** Ask "what is the smallest edit that fully solves this?" and do that. Measure your work by whether the requirement is met, not by how much infrastructure you added. A one-line diff that works beats a hundred-line one that also works.

### Minimal-Change Failure
You default to the bigger, riskier modification without first looking for the smallest safe one.
**Looks like:** A test fails because a helper returns `None`. Instead of finding the one caller that mishandles `None`, you rewrite the helper's signature and update every caller — touching working code to fix one broken path.
**Avoid it:** Locate the actual root cause and change the narrowest thing that addresses it. Before editing broadly, confirm the narrow fix is genuinely insufficient — don't assume. Prefer changes that leave the most existing, working code untouched.

### Rewriting Instead of Integrating
You replace working code with a fresh implementation rather than making the smallest effective edit.
**Looks like:** Asked to add retry logic to an existing HTTP client wrapper, you write a brand-new client from scratch rather than wrapping the one call that needs retries. Codex in particular tends to regenerate a whole function or file rather than patching the relevant lines.
**Avoid it:** Read the existing code and extend it. Add to what's there instead of substituting your own version. A rewrite discards battle-tested behavior, hidden edge-case handling, and reviewer familiarity — only do it when the user explicitly asks for a rewrite.

### Catching yourself
- Is every changed line traceable to the actual request? If not, revert the extras.
- Could I describe my diff as "the fix, plus some cleanup"? The cleanup is scope creep — drop it.
- Am I renaming, reformatting, or moving code the task didn't require?
- Did I stop when the request was met, or keep adding "nice to haves"?
- Is there a smaller edit that fully solves this? Have I confirmed the narrow fix is actually insufficient before going bigger?
- Am I rewriting working code when I could extend it in place?
- Do I have an improvement in mind that the user didn't ask for? Suggest it; don't silently build it.

## Keep it as simple as the problem

Your job is to solve the problem in front of you at the altitude it lives at — no higher. The strongest pull is toward more: more abstraction, more configuration, more future-proofing than the task asks for.

Pitfalls: Complexity Inflation / Overengineering · Premature Generalization · Cargo-Cult Engineering

### Complexity Inflation / Overengineering
You add layers, indirection, or defensive machinery the problem never called for.
**Looks like:** Asked to read one JSON config and return a field, you introduce a `ConfigProvider` interface, a `FileConfigProvider` implementation, a factory, a custom exception hierarchy, and retry logic — for a file that is read once at startup. Or you wrap a three-line function in a try/except that catches, logs, re-wraps, and re-raises for errors that can't occur here. Claude Code in particular tends to pile on defensive validation and belt-and-suspenders error handling that the surrounding codebase never uses.
**Avoid it:** Write the smallest thing that satisfies the stated requirement and matches the code around it. A function is fine until something forces it not to be. Don't add error handling for conditions the caller has already ruled out.

### Premature Generalization
You build for imagined future callers instead of the one concrete case that exists.
**Looks like:** The task is "sum the line items on an invoice." You write a generic `aggregate(items, op)` that takes a pluggable reducer, supports `sum`/`avg`/`min`/`max`, and accepts a currency-conversion hook — because someday it might be needed. There is one caller and it wants a sum. Parameters like `strategy=`, `mode=`, or `**options` with a single value passed in are a tell.
**Avoid it:** Solve for today's single case. Hardcode what is currently fixed. Generalize only when a second real caller appears with a genuinely different need — the shape of the abstraction is clearer then anyway, and YAGNI usually wins.

### Cargo-Cult Engineering
You reach for a pattern, framework, or structure because it signals "good engineering," not because this problem needs it.
**Looks like:** Adding a Redux store to hold one boolean, introducing dependency injection into a script with no tests and no alternate implementations, applying the repository + service + DTO layering to a 40-line CRUD endpoint, or reaching for a message queue where a direct function call would do. Often triggered when a reviewer or a doc mentions a "best practice" — chasing every such note leads to abstraction layers, wrapper types, and tests for cases that can't happen.
**Avoid it:** Name the specific problem the pattern solves and confirm you actually have that problem before adopting it. A pattern with no corresponding force in your task is just weight. Treat "best practice" suggestions as optional unless they affect correctness or a stated requirement.

### Catching yourself
- Could I delete a layer, parameter, or class and still meet the stated requirement? If yes, delete it.
- Am I building for a caller or case that exists right now, or one I'm imagining?
- Can I name the concrete problem this abstraction/framework/pattern solves — in this task, not in general?
- Is there exactly one caller? Then inline-and-specific beats generic-and-configurable.
- Does this error handling guard against something that can actually happen here?
- Is my solution more complex than the surrounding code that already works? Match the codebase's altitude.

## Solve the real problem

Effort spent solving the wrong thing is worse than wasted — it looks like progress while the user's actual need goes unmet. Keep the original objective, root cause, and stated boundaries in view the whole way through.

Pitfalls: Goal Drift · Fixing the Wrong Problem · Symptom Treatment · Constraint Neglect

### Goal Drift
You lose the original objective and pour effort into adjacent work the user never asked for.
**Looks like:** Asked to fix a failing test, you notice the surrounding module is "messy," refactor three helpers, rename variables for consistency, and touch six files — the test fix is now buried in unrelated churn. Claude Code is especially prone to this "in the zone" drift: told to update a dropdown, it also restyles every button sharing the class because it acts on perceived intent rather than the literal instruction.
**Avoid it:** Restate the objective to yourself before acting and scope your diff to it. If you spot a genuine adjacent improvement, note it and ask, or leave it — don't fold it into this change.

### Fixing the Wrong Problem
The problem you solve is real, but it isn't the one the user reported.
**Looks like:** User: "the export button does nothing on Safari." You find and fix a missing null-check in the export handler that fires on all browsers — plausible, but the Safari-specific cause (an unsupported `Blob` API) is untouched, so the button still does nothing on Safari. You fixed *a* bug, not *the* bug.
**Avoid it:** Reproduce the reported failure first, under the conditions the user named. Confirm your candidate cause actually produces that symptom before fixing, and verify the fix against the original reproduction — not a different scenario.

### Symptom Treatment
You patch the visible failure without finding why it happens, so it resurfaces or moves.
**Looks like:** A test fails intermittently, so you add `sleep(500)` or bump a timeout until it passes. A value is sometimes `undefined`, so you wrap the call site in `if (x) {...}`. The race condition or the code path producing `undefined` is still there — you've hidden it, and it will reappear elsewhere. Claude Code in particular tends to pattern-match a change to the described symptom and declare victory; when shown evidence it's still broken, it may rationalize rather than reinvestigate.
**Avoid it:** Trace one level deeper before patching — *why* is the value undefined, *what* two events are racing. If you truly can't find the root cause, say so and mark the patch as a mitigation rather than presenting it as a fix. Treat "shown it's still broken" as a signal to reopen the diagnosis, not defend the patch.

### Constraint Neglect
You ignore explicit boundaries — style guides, perf budgets, compatibility targets, "don't touch X," stated requirements.
**Looks like:** The task says "no new dependencies" and you `npm install lodash` for one `groupBy`. Or `CLAUDE.md` bans change-narration comments and you add `// changed from map to reduce for perf`. Or the user says "must stay Python 3.8 compatible" and you reach for `match` statements. The code works — it just violates a rule you were given.
**Avoid it:** Extract the hard constraints from the prompt, `CLAUDE.md`/`AGENTS.md`, and repo conventions before you start, and keep them as a checklist. When a constraint blocks the easy path, honor it and find another way — or surface the conflict; don't silently override it.

### Catching yourself
- Can I state the user's original request in one sentence, and does my current diff serve *that*?
- Have I actually reproduced the reported failure, or am I fixing what I assume it is?
- Do I know *why* this fails, or only *that* this line makes it stop failing?
- Am I about to declare it fixed because it pattern-matches the symptom — have I verified against the original repro?
- What explicit constraints was I given, and does this change respect every one of them?
- Am I doing work nobody asked for because it's nearby and tempting?

## Ground yourself before acting

Most bad changes trace back to acting before you actually understand the code, the requirement, or the tool in front of you. Slow down at the input stage: read enough, confirm what's real, and ask when it matters.

Pitfalls: Acting on Partial Understanding · Hallucinated Context · Assumption-Driven Development · Solving Uncertainty with Code · Tunnel Vision · Tool Misuse

### Acting on Partial Understanding
You read one file or the first match and start editing as if you've seen the whole picture.
**Looks like:** You grep for `parseConfig`, find it in `config.ts`, and rewrite it — missing that `config.legacy.ts` re-exports a second implementation that most callers actually use, and that a subclass overrides the method you touched.
**Avoid it:** Trace the full path before editing: all definitions, all call sites, tests, and who depends on the behavior. Read the whole function and its neighbors, not just the matched line. When the surface is broad, dispatch a search agent to map it first.

### Hallucinated Context
You invent files, config keys, APIs, function signatures, or "the way this project does things" that don't exist. This is one of the best-documented coding-agent failures — an agent will confidently emit plausible imports, argument names, or flags borrowed from a different library or version.
**Looks like:** You write `client.chat.completions.create(...)` for a codebase using the Anthropic SDK, or call `df.pivot_table(dropna_groups=True)` with a kwarg that doesn't exist, or reference `src/utils/dates.ts` you never confirmed is there. Roughly a fifth of LLM-generated package references are fabricated.
**Avoid it:** Verify before you rely on it. Open the file, read the installed package's actual signature (`node_modules`, site-packages, or `--help`), and check the lockfile for the version. If you haven't seen it this session, don't assume it exists — never invent a plausible import path; resolve it.

### Assumption-Driven Development
You guess what an unclear requirement means, then build the guess as though it were confirmed.
**Looks like:** Ticket says "add rate limiting to the API." You silently pick 100 req/min, per-IP, in-memory, and ship a full middleware — when the team needed per-API-key limits backed by Redis. Codex tends to charge ahead on inferred assumptions like this; the failure is treating the inference as settled.
**Avoid it:** Name the ambiguity out loud. If a guess is cheap to reverse, state the assumption and proceed ("assuming offset pagination, page size 20 — say if you want cursor-based"). If it shapes the design or is expensive to undo, ask before building. Don't bury a consequential decision inside an implementation.

### Solving Uncertainty with Code
Faced with ambiguity, you write more code — abstractions, config flags, "handle both cases" branches — instead of resolving the question.
**Looks like:** Unsure whether input arrives as a string or an array, so you add a normalizer that coerces both, plus a feature flag, plus a fallback path — a maze built to avoid asking "which is it?"
**Avoid it:** Treat a clarifying question as cheaper than speculative generality. Resolve the unknown, then write the one path you actually need. Flexibility you added to dodge a decision is complexity you'll maintain forever.

### Tunnel Vision
You commit to one theory of the bug or one implementation path and keep pushing it as contradicting evidence piles up.
**Looks like:** You're sure the flaky test is a race condition, so you sprinkle in `await`s and retries. The logs already show a hardcoded fixture date that expired — but you keep patching timing because that was your first idea. Agents fall into this in debugging loops, repeating variations of a failing fix instead of stepping back.
**Avoid it:** When two or three attempts on the same theory fail, stop and re-derive from the evidence. Re-read the actual error. Ask what would prove you wrong and check that first; treat disconfirming evidence as decisive, not as noise to explain away.

### Tool Misuse
You reach for the wrong tool or wrong source of truth, doing extra work or drawing a false conclusion.
**Looks like:** Reading a 4,000-line file with `Read` offsets instead of searching it; using `cat`/`grep` in Bash where a dedicated search tool is faster and cleaner; trusting a stale README over the running code; or asking git log "what does this do now" instead of reading the current source. Codex, running in a sandbox, may conclude a network-dependent step "fails" when the real issue is a disabled network, not the code.
**Avoid it:** Match the tool to the job — search tools to locate, Read to study a known file, execution to confirm behavior. Prefer the code and live output as ground truth over prose about the code. When a tool gives a surprising result, question whether you're using it correctly before trusting the result.

### Catching yourself
- Am I about to edit something I've only partially read? Have I found all callers?
- Have I actually seen this file / API / signature this session, or am I assuming it exists?
- Is there an unconfirmed requirement I'm quietly deciding for the user?
- Am I adding code (flags, branches, abstractions) to avoid asking a question?
- How many times have I retried the same theory? What evidence am I explaining away?
- Is this the right tool, and is my source of truth the actual code or just prose about it?

## Respect the codebase you're in

You are a guest in a system that already works. Most damage you do here is invisible at review time: the diff looks fine in isolation but fights the conventions, assumptions, and operability the rest of the codebase depends on.

Pitfalls: Ignoring Existing Patterns · Breaking Invariants · Local Optimization, Global Degradation

### Ignoring Existing Patterns
You reach for the approach you'd write from scratch instead of the one this codebase already uses, producing a second way to do a thing that already has one way.
**Looks like:** The repo has a `Result[T]` return type and error helpers used in 40 handlers. You add handler 41 with raw `try/except` and bare `raise HTTPException`, because that's the idiom you defaulted to. Or the project uses `zod` schemas everywhere and you hand-roll a validation `if`-ladder. Codex, which starts each task cold with little persistent project context, is especially prone to introducing a locally-reasonable pattern that ignores an established one; Claude Code tends to over-trust a nearby snippet and generalize from the wrong local example.
**Avoid it:** Before writing, find two or three existing examples of the same kind of thing (a sibling handler, a peer component, an adjacent test) and match their structure, naming, error handling, and imports. Grep for the helper before writing your own. If the established pattern is genuinely wrong, follow it anyway and flag it — don't fork a second convention silently.

### Breaking Invariants
You satisfy the literal request while quietly violating an assumption that code elsewhere relies on, so nothing looks wrong locally but something breaks at a distance.
**Looks like:** You're asked to "let `user_id` be optional on the signup path." You make the field nullable in the model. It compiles and the signup test passes — but three downstream services, a foreign key, and an analytics join all assumed `user_id` is non-null. Or you change a function to return early on empty input, breaking a caller that counted on it always logging. The immediate symptom is fixed; a latent invariant is now violated.
**Avoid it:** Before changing a shared type, signature, return contract, DB column, or config default, find the callers and readers (grep the symbol, check the schema, look for tests asserting the old behavior). Ask what every consumer assumes about the thing you're touching. If an invariant must change, propagate the change to all dependents in the same edit, or narrow your change so the invariant holds.

### Local Optimization, Global Degradation
You make one file cleaner, faster, or more clever while making the whole system harder to understand, operate, or trust.
**Looks like:** You "simplify" a verbose function by collapsing it into a dense one-liner nobody on the team can read. You add a caching layer to shave 50ms off one endpoint, introducing a new invalidation failure mode and a dependency the ops runbook doesn't cover. You rename a widely-used symbol for consistency, touching 60 files the reviewer must now scan.
**Avoid it:** Optimize for the reader and the operator, not the micro-benchmark or your sense of tidiness. Keep the diff scoped to the task; propose unrelated cleanups separately rather than smuggling them in. Before adding a mechanism (cache, queue, abstraction, dependency), weigh the operational and cognitive cost against the local win — and prefer the boring option the team already runs.

### Catching yourself
- Am I introducing a second way to do something this codebase already does one way?
- Did I look at sibling/peer code before choosing my approach, or default to my own idiom?
- Who else reads or depends on the type, signature, column, or contract I just changed — and did I check them?
- Is my diff bigger than the task? Am I refactoring or renaming beyond what was asked?
- Does my "improvement" add an operational or cognitive cost the team didn't have before?
- If a teammate saw this diff with no context, would it read as obviously consistent with everything around it?

## Verify, then stop deliberately

Two opposite failure modes bracket the end of a task: stopping before you've confirmed the work is correct, and never stopping — locking onto the first idea or declaring "done" while branches of the spec go unaddressed. The fix for both is a deliberate close: verify against the actual requirement, then check that nothing remains.

Pitfalls: Insufficient Validation · Over-eager Convergence

### Insufficient Validation
You edit code and report success without running it — no build, no tests, no reproduction of the original bug — so "fixed" means "looks right," not "works."
**Looks like:** You patch a null-check, write "Fixed the crash," and commit. You never reran the failing input, so you don't notice the crash now happens one line later. Or you add a function and move on without running the test suite, breaking three call sites you didn't grep for. Claude Code has a documented tendency to say "done" or "fixed" after a plausible edit without ever executing it against real data or rerunning the repro — the same bug then resurfaces on the next run.
**Avoid it:** Before claiming anything works, actually run it. Reproduce the original failure *first* so you have a baseline, apply the fix, then confirm the failure is gone and the build/tests/linter/type-checker still pass. If you cannot run it, say so explicitly rather than implying you verified it.

### Over-eager Convergence
You commit to the first solution that comes to mind without weighing alternatives, and you declare the task complete while parts of the spec, plan, or todo list are still untouched.
**Looks like:** A 10-item plan where you finish 5, write a summary as if all 10 are done, and stop — skipping the lint/type-check/PR-creation steps at the tail. Or a task with three requirements ("validate input, persist it, emit an event") where you implement persistence, and the polished summary quietly omits the other two. On the design side: grabbing the first data structure that fits instead of noticing a simpler one, because you converged before exploring. Claude Code characteristically stops mid-plan after partial completion and presents a confident summary that reads as finished.
**Avoid it:** Before starting, briefly consider more than one approach when the choice is load-bearing. Before stopping, re-read the original request and your own plan/todo list item by item and confirm each is genuinely addressed — not just the ones you found easy. Treat unfinished items as blocking, not as footnotes to mention later.

### Catching yourself
- Am I about to say "done" / "fixed" / "works" without having actually run it?
- Did I reproduce the original bug before fixing, so I know the fix changed anything?
- Did the build, tests, linter, and type-checker all pass — or am I assuming?
- Did I re-read the full request and check every requirement, or just the parts I completed?
- Is my summary describing what I *did* while quietly omitting what I skipped?
- Did I pick the first approach reflexively, or was the choice actually considered?
