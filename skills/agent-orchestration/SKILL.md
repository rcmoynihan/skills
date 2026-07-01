---
name: agent-orchestration
description: Choose and coordinate agent architectures, from a single agent with tools up to full multi-agent orchestration. Covers when one agent suffices, the core orchestration patterns (sequential, concurrent, router, group chat, handoff, hierarchical, magentic), composable techniques (reflection, best-of-N, decompose-solve-merge), agent communication, context and state management, reliability, cost, security, and result aggregation. Use when designing an agent system and deciding how many agents it needs and how they coordinate.
---

# Agent Orchestration

Design agent systems at the lowest level of complexity that reliably solves the problem, then coordinate specialized agents only when a single agent genuinely can't do the job. This skill covers when to use one agent versus many, the orchestration patterns for coordinating multiple agents, and the cross-cutting concerns (context, reliability, cost, security) that determine whether a multi-agent system actually works in production.

## Start with the least complexity that works

Agent architectures exist on a spectrum. Each step up adds coordination overhead, latency, cost, and new failure modes. Use the simplest tier that meets your requirements, and escalate only when you hit a concrete limit.

| Level | What it is | Use when | Cost |
| --- | --- | --- | --- |
| **Direct model call** | One prompt to a model. No agent loop, no tools. | Single-step tasks: classification, summarization, translation, extraction. If prompt engineering solves it, you don't need an agent. | Least complex. |
| **Single agent with tools** | One agent that reasons and chooses from available tools, knowledge sources, and APIs, looping over multiple model calls to refine results. | Varied queries within a domain, where some requests need dynamic tool use (lookups, queries, retrieval). | Often the right default. Simpler to debug and test than multi-agent, still supports dynamic logic. Set iteration limits to guard against infinite tool-call loops. |
| **Multi-agent orchestration** | Multiple specialized agents coordinated by an orchestrator or peer protocol that manages work distribution, context sharing, and aggregation. | Cross-domain problems, distinct per-agent security boundaries, or tasks that benefit from parallel specialization. | Adds coordination overhead, latency, and distributed-systems failure modes. Justified only when a single agent can't reliably handle the task. |

The core discipline: **add tools before you add agents, and add agents before you add orchestration.** Reach for the next tier only when the current one provably fails.

## Direct model calls

A direct model call is a single prompt sent to a model, returning a single response. There is no loop, no tool access, and no decision about what to do next — the model transforms input to output in one pass. This is the floor of the complexity ladder, and a large share of "AI features" belong here.

### What a direct call can do well

- **Single-step transformations:** classification, summarization, translation, extraction, rewriting, sentiment scoring, format conversion.
- **Deterministic-enough tasks** where a well-crafted prompt (with few-shot examples and a fixed output schema) reliably produces what you need.
- **Building blocks** for larger systems: individual stages of a pipeline or aggregation steps are often direct calls, not agents.

### When a direct call is the right choice

Prefer a direct call when the task completes in one pass and requires no runtime decision *by the model* about which action to take. External data is fine: retrieval-augmented generation (RAG), templated prompts, and other dynamic context assembly are all still direct calls, because *your code* fetches and assembles the prompt before the single model call. The line isn't whether external data is involved — it's whether the model itself loops or chooses what to do next. If prompt engineering, with whatever context you've assembled, solves the problem, you don't need an agent; the agent loop only adds latency, cost, and failure surface.

### When to step up to a single agent

Move to a single agent when the task needs the *model* to decide to *do* something between input and output: look up live data, call a tool or API, choose what to retrieve, or take multiple reasoning steps whose path depends on intermediate results. (Static RAG, where retrieval is a fixed step your code runs every time, stays a direct call; *agentic* retrieval, where the model decides whether and what to fetch, is an agent.) The moment the model must decide "what next" rather than just "what answer," you've crossed from a direct call into agent territory.

## Single agents

A single agent is one language model running in a loop: it reasons about a request, decides whether to call a tool or knowledge source, incorporates the result, and repeats until it has an answer. It is far more capable than a direct model call — it can do multi-step reasoning, retrieval, dynamic tool selection, and iterative self-correction — but it operates within one context window and one view of the world.

### What a single agent can do well

- **Handle a whole domain.** With a good prompt and the right tools, one agent can field the full range of requests in a single domain (e.g., order support: status lookups, returns, FAQs).
- **Use tools dynamically.** It chooses which tool to call based on the request rather than a fixed script. Protocols like MCP standardize how it discovers and invokes tools.
- **Iterate.** It can draft, check its own work, retry a failed tool call, and refine — all inside one session.
- **Specialize on demand.** Progressive disclosure ("skills": prompts, scripts, and resources loaded only when relevant) lets one agent carry many specializations without spawning separate agents. This keeps a single agent viable much further up the complexity curve.

### When a single agent is the right choice

Prefer a single agent when:

- The task fits within one domain and one manageable context window.
- The prompt and tool set stay comprehensible — you can still reason about why the agent does what it does.
- You don't need genuine parallelism, independent perspectives, or separate security boundaries.
- Debuggability, low latency, and low cost matter (one agent means fewer model calls and no cross-agent coordination).

Before escalating, exhaust the cheaper ways to make a single agent stronger: sharpen the prompt, add or refine tools, add retrieval, add progressive-disclosure skills, and add an internal reflection step (the agent critiques its own draft before finalizing). Decision-making and flow-control overhead often exceed the benefit of splitting a task across agents.

### When to graduate to multi-agent

Move to multiple agents when you hit a concrete wall that a single agent can't clear:

- **Context no longer fits.** The knowledge or the tool set is too large for one prompt, and the agent's reliability degrades (tool overload, contradictory instructions, context thrash).
- **Distributed ownership.** Multiple teams maintain distinct capabilities independently, and a single monolithic prompt becomes unmaintainable.
- **Security boundaries.** Different parts of the task require different privileges, network access, or data isolation that one agent can't safely hold at once.
- **Genuine parallelism.** Subtasks are independent and running them concurrently meaningfully cuts latency, or the problem benefits from multiple independent perspectives on the same input.

A single agent can still be infeasible for reasons unrelated to capability — security boundaries and network line-of-sight can force a split even when one agent could otherwise reason through the task.

## Why multi-agent, when it is warranted

When the task genuinely exceeds a single agent, decomposing it into coordinated specialists buys you:

- **Specialization** — each agent focuses on one domain or capability, reducing prompt and code complexity.
- **Scalability** — add or modify agents without redesigning the whole system.
- **Maintainability** — test and debug agents individually.
- **Optimization** — each agent can use a different model, approach, tools, and compute budget matched to its task.

These gains are real but conditional. They only materialize if the coordination overhead stays smaller than the complexity you removed from any single agent.

## Orchestration patterns

The patterns below are the same handful of shapes that recur across frameworks under many names. They sit on a spectrum from **deterministic** (the workflow decides who runs next) to **dynamic** (the agents or a manager decide at runtime).

### Sequential (pipeline)

*Also known as: prompt chaining, linear delegation, pipes-and-filters.*

Agents run in a fixed, predefined order; each consumes the previous agent's output. Routing is deterministic — the next agent is defined by the workflow, not chosen by an agent.

```
Input → Agent 1 → Agent 2 → ... → Agent n → Result
        (shared/accumulating state spans the pipeline)
```

- **Use when:** stages have clear linear dependencies; progressive refinement (draft → review → polish); data-transformation pipelines where each stage adds value the next depends on; stages that can't be parallelized.
- **Avoid when:** stages are embarrassingly parallel; the workflow needs backtracking, iteration, or dynamic routing on intermediate results; agents need to collaborate rather than hand off.
- **Strengths:** simple, predictable, auditable.
- **Weaknesses:** no parallelism; **errors in early stages propagate and compound** downstream.
- **Example:** contract generation — template selection → clause customization → compliance review → risk assessment.

### Concurrent (parallel, fan-out/fan-in)

*Also known as: scatter-gather, map-reduce, ensemble.*

Multiple agents run simultaneously on the same input, each contributing an independent perspective or specialization. Results are typically aggregated, though aggregation isn't required — agents may each act independently (e.g., write to different stores).

```
              ┌→ Agent 1 →┐
Input → split ┼→ Agent 2 →┼→ aggregate → Result
              └→ Agent n →┘
```

- **Use when:** subtasks parallelize cleanly; you want multiple independent perspectives (brainstorming, ensemble reasoning, quorum/voting); latency-sensitive scenarios where parallelism cuts wall-clock time.
- **Avoid when:** agents must build on each other's work; order matters for reproducibility; you can't safely coordinate concurrent writes to shared state; there's no clear conflict-resolution strategy when results disagree.
- **Aggregation strategies:** vote / majority-rule for classification; weighted merge for scored recommendations; a model-synthesized summary to reconcile results into one narrative.
- **Strengths:** best parallelization; broad coverage of the problem space; lower latency.
- **Weaknesses:** resource-intensive (simultaneous model calls can spike quota); requires an explicit conflict-resolution strategy.
- **Example:** evaluate one stock with parallel fundamental, technical, sentiment, and ESG agents, then combine into a recommendation.

### Router (stateless parallel dispatch)

*A specialized concurrent pattern.*

A router decomposes the query, invokes zero or more specialized agents in parallel, and synthesizes their results into a coherent response. Typically **stateless** — each request is handled independently, giving consistent per-request cost but no memory across turns.

- **Use when:** distinct verticals or knowledge domains that a query may span; you need parallel cross-source execution; enterprise knowledge bases and multi-vertical assistants.
- **Avoid when:** you need multi-hop reasoning across turns or conversation history (statelessness means repeated routing overhead).
- **Strengths:** excellent parallelization; consistent per-request performance; good distributed development.
- **Weaknesses:** no multi-hop; no state carry-over between turns.

### Group chat (debate, council)

*Also known as: roundtable, collaborative, multi-agent debate.*

Multiple agents collaborate in a shared conversation thread. A chat manager decides who speaks next and manages the mode, from free brainstorming to structured approval gates. Agents are usually read-only (they discuss; they don't change external systems). Everything lands in one accumulating thread, giving transparency and auditability, and the pattern naturally supports a human acting as, or overriding, the chat manager.

- **Use when:** problems solved through discussion — collaborative ideation, consensus-building, debate, multidisciplinary dialogue; structured QA, compliance review, or editorial create-then-validate workflows.
- **Avoid when:** a simple pipeline suffices; real-time needs make discussion overhead unacceptable; the chat manager has no objective way to decide the task is complete.
- **Strengths:** consensus and quality control; transparent, auditable thread; natural human-in-the-loop.
- **Weaknesses:** conversation loops and control difficulty — **keep to three or fewer agents** to stay controllable.
- **Example:** specialist agents (community, environmental, budget) debate a park proposal to anticipate feedback before public review.

#### Maker-checker loop (a group-chat sub-pattern)

*Also known as: evaluator-optimizer, generator-verifier, critic loop, reflection loop.*

One agent (the maker) proposes; another (the checker) evaluates against defined criteria and pushes back with specific feedback; the maker revises and resubmits. The loop repeats until the checker approves or an iteration cap is reached.

```
maker → output → checker →(reject + feedback)→ maker → ...
                        └─(approve)→ done
                        └─(cap reached)→ fallback / escalate to human
```

Requires **clear acceptance criteria**, an **iteration cap**, and a **defined fallback** (escalate to a human, or return the best-so-far with a quality warning). Without these it never terminates. The same loop, collapsed into one agent that critiques its own draft, is a reflection step — reach for that before spawning a separate checker.

### Handoff (dynamic delegation)

*Also known as: routing, triage, transfer, dispatch.*

One active agent at a time. Each agent assesses the task and either handles it or transfers full control to a more appropriate agent (via a tool call). The right agent isn't known upfront — it emerges during processing. Handoffs are stateful: state persists across turns as the active agent changes, enabling staged sequential workflows.

```
Agent A ──(recognizes limit)──→ Agent B ──(recognizes limit)──→ Agent C → human
```

- **Use when:** the right specialist can't be predetermined; expertise requirements surface mid-conversation; multi-domain problems handled by one specialist at a time; you can define signals for "this agent has hit its limit → hand to that one."
- **Avoid when:** the right agent (or sequence) is identifiable from the initial input — use deterministic dispatch/classification instead; multiple operations should run concurrently; you can't bound handoff loops.
- **Strengths:** fluid multi-turn conversation with natural context carry-over; strong multi-hop; excellent direct user interaction.
- **Weaknesses:** no parallelism; **risk of infinite handoff loops** and unpredictable routing paths.
- **Example:** a support triage agent hands network issues to an infrastructure agent, billing disputes to a financial agent, and unresolvable cases to a human.

### Hierarchical (supervisor and subagents)

*Also known as: centralized orchestration, subagents-as-tools, orchestrator-worker.*

A supervisor agent coordinates specialized subagents, often by calling them as tools: it decides which to invoke, with what input, and how to combine their results. Subagents are typically stateless with strong context isolation, and results flow back through the supervisor.

```
                 ┌→ Subagent 1 ┐
User → Supervisor ┼→ Subagent 2 ┼→ Supervisor synthesizes → Result
                 └→ Subagent n ┘
```

- **Use when:** multiple distinct domains under central control; subagents needn't converse with the user directly; a lead agent delegating to domain experts (e.g., research decomposed across specialists).
- **Avoid when:** the extra routing hop isn't worth it, or subagents need direct back-and-forth with the user.
- **Strengths:** excellent distributed development and parallelization; strong multi-hop; **context isolation prevents token bloat** because each subagent works in its own clean context.
- **Weaknesses:** one extra model call per interaction (results round-trip through the supervisor); limited direct user interaction; added latency from centralized routing.

### Magentic (adaptive planning)

*Also known as: dynamic orchestration, task-ledger-based orchestration, adaptive planning.*

Built for open-ended problems with no predetermined solution path. A manager agent builds and continuously refines a **task ledger** (goals, subgoals, statuses) by consulting specialized agents. It iterates, backtracks, reorders, and adds or removes tasks as context evolves, repeatedly checking whether the goal is met or stalled. The specialized agents here typically hold tools that change external systems, so the pattern emphasizes building and documenting the plan as much as executing it. It is, in effect, group chat where a manager drives toward a plan and the other agents act on the world.

```
Manager builds task ledger → invokes agents → evaluate goal loop
      ↑                                              │
      └────────── refine ledger ←─(not done)─────────┘
                                  └─(done)→ Result
```

- **Use when:** no predetermined path; the plan must be assembled from multiple specialists' input; you want a reviewable, auditable plan (human approval before/after execution); agents wield tools with real external effects.
- **Avoid when:** the path is deterministic; the task is simple; work is time-sensitive (this pattern optimizes for a viable plan, not speed); you can't bound stalls or loops.
- **Strengths:** handles ambiguity and emergent structure; produces an adaptive, auditable plan.
- **Weaknesses:** slow to converge; can stall on ambiguous goals; **least predictable cost** (the manager iterates until it has a viable plan).
- **Example:** SRE incident remediation that invents a plan live and pivots strategy (e.g., from deployment rollback to database recovery) as diagnostics come in.

### Pattern selection at a glance

| Pattern | Coordination | Routing | Best for | Watch out for |
| --- | --- | --- | --- | --- |
| **Sequential** | Linear pipeline | Deterministic, predefined | Step-by-step refinement with clear dependencies | Early-stage failures propagate; no parallelism |
| **Concurrent / Router** | Parallel on same input | Deterministic or dynamic selection | Independent multi-perspective analysis; latency-sensitive | Needs conflict resolution; resource-intensive |
| **Group chat** | Shared thread | Chat manager controls turns | Consensus, brainstorming, maker-checker validation | Conversation loops; hard past ~3 agents |
| **Handoff** | One active agent | Agents decide transfers | Right specialist emerges during processing | Infinite handoff loops; unpredictable paths |
| **Hierarchical** | Supervisor + subagents | Supervisor delegates | Central control over distinct domains; context isolation | Extra round-trip per call; weak direct user interaction |
| **Magentic** | Plan-build-execute ledger | Manager assigns/reorders dynamically | Open-ended problems with no set path | Slow to converge; unpredictable cost |

## Composable techniques

These layer on top of the patterns above rather than replacing them.

- **Decompose-solve-merge.** A planner splits a problem into scoped subtasks, dedicated agents solve each with focused context, and an integrator merges the outputs. Keeps individual sessions manageable; the backbone of most hierarchical and sequential systems.
- **Speculative execution (best-of-N).** Run several independent attempts at the same problem in parallel, then select the best by an evaluation criterion. Trades cost for quality and robustness when the solution space is wide.
- **Reflection / self-critique.** An agent (or a paired checker) evaluates a draft against a rubric and revises. The single-agent version is the cheapest reliability win available; the paired version is the maker-checker loop.
- **Shared state / blackboard.** Agents read from and write to a shared state object (decisions, attempts, constraints, results) instead of passing everything through messages. Decouples agents and enables resumption, at the cost of needing careful concurrency control.
- **Progressive disclosure (skills).** A single agent loads specialized instructions and resources on demand. Captures much of the distributed-development benefit of multi-agent without the coordination cost — often the right move before splitting into agents.

## Choosing a pattern

1. **Can a direct call or single agent do it?** If yes, stop there. Strengthen the single agent (prompt, tools, retrieval, skills, reflection) before splitting.
2. **What forces multi-agent?** Context limits, distributed ownership, security boundaries, or genuine parallelism. If none apply, don't.
3. **Is the flow deterministic or dynamic?**
   - Deterministic order → **sequential**.
   - Independent work on the same input → **concurrent / router**.
   - Right agent emerges at runtime → **handoff**.
   - Discussion / consensus / validation → **group chat** (+ maker-checker).
   - Central control over distinct domains → **hierarchical**.
   - Open-ended, plan-as-you-go → **magentic**.
4. **Do stages differ?** Combine patterns — see below. Don't force one workflow into a single pattern.

## Agent communication and state

How agents exchange information shapes both reliability and cost.

- **Direct messaging.** Agents pass structured messages to each other. Simple, but couples agents tightly and grows context fast.
- **Shared state / blackboard.** Agents coordinate through a shared store rather than direct messages. Loosely coupled and resumable; requires concurrency discipline.
- **Manager-mediated.** A coordinator brokers all communication (broadcasts, routing, aggregation). Centralizes control at the cost of a routing hop.

### Context and state management

Context windows grow fast in multi-agent systems — each agent adds its own reasoning, tool results, and intermediate output. This is the single most common source of degradation.

- **Decide what the next agent actually needs.** Sometimes the full raw context; often a compacted summary; sometimes just a fresh instruction set. Don't forward context that doesn't help the receiving agent.
- **Compact and prune between agents.** Summarize or selectively drop content at each handoff to stay within model limits and avoid quality degradation. Unpruned context accumulates errors and contradictions (context pollution) that mislead later agents.
- **Persist state externally for long-running work.** Store task progress, intermediate results, and history in a durable store so agents can resume after interruptions. Scope persisted state to the minimum necessary for token and privacy reasons.

## Cross-cutting concerns

Multi-agent systems are distributed systems with nondeterministic components. These concerns determine whether one works in production.

### Reliability

- Implement timeouts, retries, and circuit breakers for agent dependencies.
- Provide graceful degradation when one agent faults, rather than failing the whole workflow.
- **Surface errors instead of hiding them** so downstream agents and the orchestrator can respond.
- **Validate each agent's output before passing it on** — malformed, low-confidence, or off-topic output cascades through a pipeline. Retry, request clarification, or halt.
- Isolate agents so they don't share single points of failure (compute isolation; watch for a shared model endpoint or knowledge store becoming a rate-limit bottleneck under parallelism).
- Use checkpointing to recover interrupted orchestrations after a fault or redeploy.

### Cost

Multi-agent multiplies model invocations; each agent spends tokens on instructions, context, reasoning, and tool calls.

- **Right-size the model per agent.** Classification, extraction, and formatting agents can use smaller, cheaper, faster models with no loss in overall quality.
- **Monitor tokens per agent and per run** to find the expensive agents and target optimization.
- Compact context between agents to cut token volume.
- Note the cost profile per pattern: sequential and handoff accumulate cost step by step; concurrent can spike consumption; magentic is the least predictable because the manager iterates until it has a plan.

### Security

- Authenticate and secure inter-agent communication.
- Apply least privilege per agent.
- **Security-trim in every agent.** Agents often need broad access to knowledge stores to serve all users, but each must not return data the current user can't see.
- Apply content-safety guardrails at multiple points — user input, tool calls, tool responses, and final output — since intermediate agents can introduce or propagate harmful content.
- Design audit trails to meet compliance requirements.

### Observability and testing

- Instrument every agent operation and handoff; troubleshooting a distributed AI system requires it.
- Track per-agent performance and resource metrics to baseline, find bottlenecks, and optimize.
- Design testable interfaces for individual agents.
- Test individual agents and the system as a whole. Because outputs are nondeterministic, use scoring rubrics or model-as-judge evaluations rather than exact-match assertions.

### Human-in-the-loop

Humans participate as observers in group chat, reviewers in maker-checker loops, and escalation targets in handoff and magentic orchestrations.

- Identify which points need human input, and whether it's optional or mandatory.
- Decide whether the human response is an approval (advance the workflow) or feedback (loop back for refinement).
- Mandatory gates make that step synchronous — persist state there so the orchestration resumes without replaying prior work.
- Scope gates to specific tool invocations (approve only sensitive actions) so low-risk work proceeds autonomously.

## Antipatterns

- Using a complex pattern where a single agent or basic sequential/concurrent orchestration would suffice.
- Adding agents that provide no meaningful specialization.
- Ignoring the latency of multi-hop communication.
- Sharing mutable state between concurrent agents, producing transactionally inconsistent data.
- Using a deterministic pattern for an inherently dynamic workflow, or a dynamic pattern for an inherently deterministic one.
- Ignoring resource constraints when choosing concurrent orchestration.
- Letting context windows balloon as agents accumulate information unchecked.

## Combining patterns

Most real systems mix patterns across stages. Use sequential orchestration for initial data processing, then switch to concurrent for parallelizable analysis, then a maker-checker loop for final validation. Don't force one workflow into a single pattern when different stages have different characteristics — the orchestration layer is where you compose them.

## Coding agents and harness engineering

Coding is the most demanding application of orchestration, and the discipline of structuring coding agents reliably has its own name: **harness engineering** — building the scaffolding around agent coding sessions that defines when each agent runs, what context it receives, how its output is validated and passed on, and how errors are handled. It sits one layer above prompt engineering (the right instructions) and context engineering (the right information within one session): the harness coordinates *across* sessions and agents to turn probabilistic, one-off generations into repeatable pipelines.

Two properties make coding unusual among agent domains, and both shape how you orchestrate it: the work is **tightly coupled** (edits depend on other edits and share context), and correctness is **checkable** (tests, builds, and linters give ground truth).

### The orchestrator-worker spine

The dominant coding shape is orchestrator-worker: a **lead** owns planning and synthesis — it decomposes the task, spawns **workers** with narrow mandates in isolated contexts, and merges their results. Workers report back to the lead; they don't converse with the user or, usually, each other.

The primary reason this pattern exists for coding is **context preservation, not raw speed**. A model's context window fills fast and quality degrades as it fills — a single codebase exploration or debugging session can burn tens of thousands of tokens on grep dumps, file reads, and test logs that are useless once the answer is extracted. Running that work in a worker's isolated context keeps the verbose, non-reusable output out of the lead's window, so the lead stays clear for planning and integration. Isolation buys two more things almost for free: **per-worker permission scoping** (a reviewer gets read-only tools; a coder gets edit access) and **unbiased review** (a reviewer in a fresh context sees only the diff and the criteria, not the reasoning that produced the change, so it judges the result on its own terms).

### Match the model to the role (tiering)

Size each role's model to its cognitive load rather than running everything on the strongest model:

- **Strong/frontier model** for planning, decomposition, ambiguity and requirements resolution, architecture, cross-result synthesis, and final review — the genuinely hard reasoning.
- **Mid-tier model** for scoped implementation: well-defined tasks, standard patterns, first drafts against a clear spec.
- **Cheapest, fastest model** for high-volume, read-mostly, mechanical work: codebase search, extraction, formatting, classification.

The leverage comes from an asymmetry: **a strong, well-specified plan lets a weaker model execute reliably** — the intelligence lives in the decomposition and the harness, not in every worker. This is also what makes fan-out affordable, since parallel workers multiply token cost (multi-agent systems can use on the order of 15× the tokens of a single chat) and a cheaper per-worker tier offsets the multiplier. Two failure modes bound the choice: **under-powering the planner** is the expensive mistake, because a bad decomposition produces subtasks that don't compose and the error propagates to every worker; **over-powering trivial workers** burns the multiplier for no quality gain. Add **escalation** so a low-confidence or malformed worker result retries on a stronger tier. Harnesses typically expose this as a plan-on-strong-then-execute-on-cheaper mode, or per-worker model pinning (e.g., read-only search on the cheapest tier).

### Fan-out: what parallelizes safely

Coding fan-out is safe and encouraged for **independent, read-mostly work**, and risky for parallel writing:

- **Safe:** parallel exploration (one worker per subsystem, then synthesize), parallel review (independent lenses — one for security, one for performance, one for test coverage, on the same diff), and batch fan-out over a list of independent files.
- **Risky:** parallel implementation. Independent coders that can't see each other's work produce conflicting or incoherent edits — the canonical failure is two agents building mismatched halves of one feature. Only genuinely independent work with clean, non-overlapping file boundaries should run in parallel.

```
                ┌→ Worker A ─┐
Lead → fan out ─┼→ Worker B ─┼→ compact results → integrate → Result
                └→ Worker C ─┘
   safe when workers own disjoint files and only read/explore
```

When you do parallelize writes: **partition by file ownership** (each worker owns a disjoint set of files), **isolate edits in per-worker git worktrees or branches** so they can't collide, and **map dependencies first** so anything ordered or shared runs sequentially. Watch the fan-in too — many detailed worker summaries can re-fill the lead's context, so compact results on the way back up. A practical default is **3–5 workers per level**; three focused workers usually beat five scattered ones.

### Hierarchical nesting: leads and workers

For work larger than one level of fan-out, nest: the main orchestrator spawns **feature leads** (one per slice of the work), and each lead decomposes its slice into **workers**, runs its own draft → review → revise loop, and returns only reviewed, passing code up to the orchestrator. The orchestrator integrates a few leads' summaries; it never sees the fine-grained decomposition each lead manages.

```
Orchestrator
├─ Lead (slice 1)
│   ├─ Worker ─┐
│   ├─ Worker ─┼→ draft → review → revise   (leaf loop)
│   └─ Worker ─┘
└─ Lead (slice 2) → …
   each leaf loops to green · each lead synthesizes one summary · orchestrator integrates
```

Nest for the same reason you isolate workers — **context hygiene**. Spawning six workers directly fragments the orchestrator's context; spawning two leads that each own three workers keeps the top level talking to only two agents while pushing the detail down a level. A lead should **subdivide further** when its subtasks span fundamentally different domains, when its worker count would exceed roughly five to ten, or when it can no longer track all the context itself; otherwise it should just do the work with a flat set of workers.

Results **roll up** level by level: each leaf runs its maker-checker loop, each lead synthesizes its workers' output into a single summary, and the orchestrator synthesizes the leads'. **Bound the depth** — roughly two to three levels handles almost anything, and each additional level compounds latency, token cost, context dilution, error propagation, and loss of oversight. Two hard bounds keep nesting from running away: **cap refinement-loop iterations** (commonly two or three) with a defined fallback, and **prevent leaf workers from delegating further** so the tree can't recurse without limit.

### Verification: give the agent a check it can run

Coding's checkability is its biggest orchestration advantage — use it. The strongest coding wins are **asymmetric**: one generator plus an independent verifier, not N symmetric coders. Give the agent a check it can run and it closes its own loop instead of leaving you as the verification step:

- **Runnable checks**, in ascending strength: run-and-iterate in one prompt → a goal condition re-checked by an evaluator after each turn → a hard gate (e.g., a stop hook) that blocks completion until tests, lint, and build pass → an **adversarial second opinion**, where a fresh model tries to refute the result so the agent doing the work isn't the one grading it.
- **Test-driven validation:** have one agent write tests, then another write code to pass them.
- **Separate exploration and planning from implementation** (read-only research first) so the agent solves the right problem, then verify the implementation against the written plan.

One caution: a reviewer instructed to find gaps will almost always report some, even when the work is sound. Scope reviewers to correctness and requirements, or chasing every reported "gap" turns into over-engineering.

### When not to split a coding task

Multi-agent coding is not the default — it is a specific tool for a subset of coding work. Because most coding is tightly coupled (edits depend on other edits, everything shares context), it is often a **poor fit** for the symmetric multi-agent decomposition that works so well for embarrassingly-parallel research. Keep it a single agent for quick targeted changes, tightly sequential work, same-file edits, and any task where each step must reason about all the previous ones.

Reach for multiple agents when the work is genuinely separable (independent slices with clean file boundaries), read-mostly and parallel (exploration, review), or high-value enough to justify the token multiplier — and above all for the asymmetric **generator-plus-verifier** split, which pays off even on work that isn't parallel at all. And remember that **harness reliability often dominates model choice**: a dependable edit mechanism, clean context per agent, runnable checks, iteration caps, and clear loop-termination conditions frequently matter more to the outcome than which model you run.

## Getting started

1. **Prove you need more than one agent.** Try a direct call, then a single agent with tools. Escalate only on a concrete limit (context, ownership, security, parallelism).
2. **Start small.** Two or three agents with a clearly documented flow.
3. **Give each agent a sharp definition.** Role, goal, tools, and the minimum context it needs.
4. **Choose a pattern deliberately** using the decision guide, and combine patterns where stages differ.
5. **Build in the cross-cutting concerns from the start** — output validation, iteration caps, context compaction, per-agent model sizing, and observability.
6. **Test agents individually and together** with rubric- or judge-based evaluation, then iterate.

## Implementation checklist

- [ ] Confirmed a direct model call (with RAG or dynamic prompting) can't do the job
- [ ] Confirmed a single agent (with tools, retrieval, skills) can't do the job
- [ ] Defined each agent's role, goal, expertise, and tools
- [ ] Chose an orchestration pattern (and any pattern combination) deliberately
- [ ] Defined communication and shared-state approach
- [ ] Set success criteria and acceptance rules for each task
- [ ] Set iteration caps and fallback/escalation for all loops
- [ ] Planned context compaction/pruning between agents
- [ ] Right-sized the model for each agent
- [ ] Added output validation before each handoff
- [ ] Added timeouts, retries, graceful degradation, and checkpointing
- [ ] Applied least privilege and security trimming per agent
- [ ] Instrumented per-agent and end-to-end observability
- [ ] Identified human-in-the-loop gates and persisted state at them
- [ ] Evaluated with rubrics/judge, not exact-match assertions

## Resources

### Frameworks

- **LangGraph**: https://langchain-ai.github.io/langgraph/
- **Microsoft Agent Framework** (and Semantic Kernel): https://learn.microsoft.com/en-us/agent-framework/overview
- **CrewAI**: https://crewai.com/
- **AutoGen**: https://microsoft.github.io/autogen/
- **OpenAI Agents SDK / Swarm**: https://openai.github.io/openai-agents-python/multi_agent/
- **PydanticAI**: https://ai.pydantic.dev/

### Harness subagent support

- **Claude Code**: https://code.claude.com/docs/en/agents
- **Codex CLI**: https://developers.openai.com/codex/subagents
- **OpenCode**: https://opencode.ai/docs/agents/
- **pi.dev**: https://pi.dev/packages/pi-subagents

### Further reading

- Azure Architecture Center — AI agent orchestration patterns: https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns
- LangChain — choosing a multi-agent architecture: https://www.langchain.com/blog/choosing-the-right-multi-agent-architecture
- Anthropic — How we built our multi-agent research system: https://www.anthropic.com/engineering/multi-agent-research-system
- Cognition — Don't build multi-agents: https://cognition.com/blog/dont-build-multi-agents
