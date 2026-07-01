---
name: prompt-engineering
description: Write strong prompts for LLMs — direct model calls, single tool-using agents, and multi-agent systems. Covers the core stance (specificity and completeness over brevity), prompt anatomy and ordering, the high-leverage techniques (be specific, give context and the why, positive instructions, multishot examples, let the model think, XML tags, system/role prompts, output formats and structured outputs, prompt chaining, guardrails), how prompting changes across execution modes (direct call vs single agent vs orchestrator vs worker), and iterating empirically with evals. Use when authoring, reviewing, or debugging a prompt or system prompt for any LLM feature or agent.
---

# Prompt Engineering

A prompt is a **specification**, not a wish. The model does what you actually said, not what you meant — so the job is to say it completely, concretely, and in the order the model reads it. The single most reliable predictor of a bad output is a vague, under-specified prompt; the cheapest fix is almost always to add the missing specificity. Write the prompt you would hand a smart, capable new colleague who is starting cold, cannot ask you follow-up questions, and will do *exactly* what the page says. Anthropic's docs make this the literal test: *"Show your prompt to a colleague with minimal context on the task and ask them to follow it. If they'd be confused, Claude will be too."*

**The through-line: prefer specificity and completeness over brevity.** Say what you want, why you want it, what "good" looks like, and what to do at the edges. Detail that removes ambiguity is the entire point of the exercise. The only things you trim are *dead weight* — redundancy, contradictions, and rules no real input would ever trigger — never the detail that tells the model what to do.

This skill is provider-general but tuned to Anthropic's guidance and current Claude models; it flags where major providers disagree. For deciding *how many agents* a system needs and how they coordinate, see the `agent-orchestration` skill — this skill is about the prompts those agents run.

## The core stance: specificity beats brevity

The instinct to keep prompts short is usually wrong. Short prompts fail because they leave the model to guess — at scope, format, audience, edge-case handling, and what "good" means — and it guesses differently every run, which reads as inconsistency and "the model is dumb." The fix is rarely a cleverer phrasing; it is *more of the right information*. Modern models make this matter *more*, not less: newer models (GPT‑4.1, current Claude) follow instructions more literally and infer less liberally, so an under-specified prompt gets you exactly the gap you left, and "a single sentence firmly and unequivocally clarifying your desired behavior is almost always sufficient to steer the model on course."

Two failure modes sit at opposite ends, and **under-specification is by far the more common one**:

- **Vague / under-specified (the usual problem):** "Summarize this." "Make it better." "Analyze the data." The model invents the length, audience, format, and criteria. Every omission is a decision you handed to the model.
- **Bloated / over-constrained (the rarer problem):** contradictory rules, the same instruction restated five ways, rules for inputs that never occur, or duplicating guidance another layer already supplies. This dilutes attention and can actively *degrade* quality.

The resolution is not "write long" or "write short" — it is **make every token earn its place, and spend as many as the task needs.** Add detail that changes the output; cut text that doesn't. When in doubt, err toward *more* specificity, because the failure you'll actually hit is the model guessing, not the model drowning in a precise instruction.

**Specificity is not shouting or padding.** Precise detail is the goal; volume, emphasis, and redundancy are not. On current models, `CRITICAL: You MUST always use this tool!!!` and hedges like "if in doubt, use X" tend to *overtrigger* and backfire — plain, specific phrasing ("Use this tool when the user asks about their order status") works better. All-caps, bribes, and threats are unnecessary; start without them. Detail earns its place by disambiguating, not by turning up the volume.

**The cut test (for trimming, not for starting):** for each rule or constraint, can you describe a realistic input that would trigger it and a distinct output it produces? If not, it's dead weight — cut it. This removes untriggerable rules and redundancy. It does **not** license replacing a concrete instruction with a vague one.

### Be specific — worked contrasts

Vague instructions get vague, inconsistent results. Describe the desired output in detail. If you want above-and-beyond behavior, ask for it explicitly rather than hoping the model infers it.

| Vague (model guesses) | Specific (model complies) |
| --- | --- |
| "Summarize this article." | "Summarize this article in 3–5 bullet points for a busy executive. Each bullet ≤ 20 words, leads with the finding, and includes the number if the article gives one. No preamble." |
| "Be friendly and professional." | "Warm, first-person, plain language. No corporate jargon. Write like a knowledgeable coworker, not a press release." |
| "Extract the key info." | "Return JSON: `{company: string, amount_usd: number, date: 'YYYY-MM-DD'}`. If a field is absent, use null — never guess." |
| "Analyze the data and give recommendations." | "1) State the single biggest driver of the change. 2) Give 2–3 recommendations, each with expected impact and risk. 3) Flag any assumption you had to make. Lead with an executive summary of ≤ 2 sentences." |
| "Create a dashboard." | "Create an analytics dashboard. Include as many relevant features and interactions as possible. Go beyond the basics to a fully-featured implementation." |

Notice what "specific" adds: **format, length, audience, ordering, tie-breaking rules, and explicit edge-case behavior** — exactly the things a short prompt leaves the model to invent.

## Anatomy of a strong prompt

Most strong prompts assemble the same components. Not every prompt needs all of them, but run down this list and add whatever removes ambiguity:

1. **Role / persona** — who the model acts as, with the specific expertise that matters ("a staff security engineer reviewing for injection and auth flaws," not "a helpful assistant"). Even a single sentence of role setting measurably shifts vocabulary, depth, and priorities.
2. **Task** — the objective, as a direct instruction. One clear "what to produce." Prefer an explicit action ("Rewrite this function to run faster") over a hedge ("Can you suggest some changes?").
3. **Context and the *why*** — background the model needs, *and the reason for the task*. Telling the model why (who reads the output, what it feeds into, what's at stake) lets it make better judgment calls on everything you didn't spell out. Anthropic's example: `NEVER use ellipses` works better as `Your response will be read aloud by a text-to-speech engine, so never use ellipses — it won't know how to pronounce them.` The model generalizes from the rationale.
4. **Detailed instructions** — the rules, steps, and criteria. Use numbered lists when order or completeness matters. Prefer positive instructions ("do X") over prohibitions.
5. **Examples** — one or more input→output demonstrations covering the normal case and the tricky ones. Usually the highest-leverage thing you can add.
6. **Output format** — the exact shape: schema, template, headings, length. Show it; don't just describe it.
7. **Constraints and edge cases** — hard limits, what to do when data is missing, ambiguous, or out of scope, and how to signal uncertainty.
8. **Reasoning space** — an explicit place/instruction to think before answering, when the task needs it.
9. **The input data** — the actual content to operate on, clearly delimited, placed **last** in long-context prompts (see ordering).

### Ordering and delimiters

Order matters, and so does marking your boundaries.

- **Put long/reference content near the top, the query/instruction near the bottom.** Models degrade at retrieving information buried in the *middle* of a long context (the "lost in the middle" effect — performance is highest when the relevant text is at the start or end). Anthropic reports that placing long documents first and the query at the end can improve response quality **by up to 30%** on complex, multi-document inputs. **Provider nuance:** OpenAI's GPT‑4.1 guidance instead recommends putting the *instructions* at **both** the top and the bottom of a long context; if only once, above the data. When in doubt for very long prompts, restate the key instruction at the end regardless.
- **Delimit every distinct part.** Use XML-style tags (`<document>`, `<example>`, `<instructions>`, `<data>`) or clear headers to separate instructions from data from examples. This stops the model confusing your data for your instructions (and blunts a class of prompt injection), and lets you *refer back* to a section by name ("using the rubric in `<criteria>`…"). Tags are Anthropic's recommended default; pick descriptive names and stay consistent. (For delimiter *style*, OpenAI found Markdown a good default and XML strong; large JSON blobs of documents performed worst.)
- **Front-load the most important instruction**, and for long prompts restate it at the end.
- **For long documents, ask the model to quote first.** "Quote the parts of the documents relevant to the question in `<quotes>`, then answer." Pulling the relevant passages before reasoning cuts through the noise of everything else.

## The high-leverage techniques

Roughly ordered by leverage — start at the top; most quality problems are solved before you reach the bottom. (This ordering is editorial guidance, not a ranking any provider publishes.)

### 1. Be clear, direct, and specific
The foundation, covered above. Say exactly what to produce, for whom, in what format, how long, and what to do at the edges. If a human colleague could misread it, rewrite it.

### 2. Give context and the *why*
Explain the purpose, the audience, and how the output will be used. "This goes into a customer-facing email, so no internal codenames" shapes more of the output than any single rule. Context lets the model resolve the thousand small ambiguities you didn't anticipate — in the direction you'd want. Explaining *why* a rule matters beats stating the rule bare.

### 3. Prefer positive instructions over negations
Tell the model what **to do**, not just what to avoid. "Respond in flowing prose paragraphs" beats "Don't use bullet points." Negations are easy to violate and often draw attention to the very behavior you're suppressing — a classic failure is a prompt saying `DO NOT ASK FOR PERSONAL INFORMATION` that makes the model do exactly that. When a boundary is essential, pair it with the positive alternative: "For billing questions, direct the user to support@example.com" rather than only "Don't answer billing questions."

### 4. Show examples (multishot)
Examples are one of the most reliable ways to steer output format, tone, and structure — the model pattern-matches on demonstrated behavior. This is the highest-leverage lever after basic specificity.

- **Include 3–5 examples** for most tasks; more for hard or highly formatted ones. Even one well-chosen example ("one-shot") sharply improves format adherence over zero.
- **Cover the tricky cases, not just the easy one** — the ambiguous input, the edge case, the "when in doubt do this" case. Examples are where you teach judgment.
- **Match the format you want back exactly.** The model imitates the shape of your examples, including whitespace, casing, and structure. Wrap them in tags (`<example>`, multiple in `<examples>`) so they're unambiguously examples, not data.
- **Keep examples relevant and diverse.** Mirror your real inputs; vary them enough that the model doesn't latch onto an incidental shared pattern. A wrong or off-distribution example ("example pollution") teaches the wrong thing.
- **Keep rules and examples consistent** — any behavior a demonstration relies on should also be stated in the instructions. You can even ask the model to critique your examples for relevance and diversity, or generate more.

### 5. Let the model think (reasoning space / chain of thought)
For anything requiring analysis, math, multi-step logic, or weighing options, give the model room to reason *before* it commits to an answer. Reasoning improves accuracy; skipping it forces a single-pass computation.

- **On reasoning-capable models, prefer general instructions over a prescribed step list.** With native/extended ("adaptive") thinking, a prompt like "think thoroughly before answering" often beats a hand-written step-by-step plan — the model's own reasoning frequently exceeds what you'd prescribe. Over-scripting the steps can *cap* it.
- **On non-reasoning models or with thinking off, use explicit structured CoT.** Tell it *what* to reason about and where: "In `<thinking>`, list the constraints, evaluate each option against them, then give your answer in `<answer>`." Naming the steps beats a bare "think step by step."
- **Separate reasoning from the answer** (`<thinking>` vs `<answer>`) so you can parse the deliverable cleanly — and, for evals, discard the reasoning.
- **Add a self-check** for tasks with a checkable answer: "Before you finish, verify your result against [the original constraints] and correct it if it fails." Cheapest reliability win available.
- **Self-consistency** for high-stakes reasoning: sample several independent reasoning paths and take the majority answer.
- Reasoning costs tokens and latency — use it where correctness matters, skip it for trivial transforms.

### 6. Use XML tags and delimiters
Covered under ordering. Structure the prompt so the model can tell instructions from data from examples from output, and so you can reference parts by name. Consistent, descriptive tag names matter more than which names you pick; nest them when content has a natural hierarchy.

### 7. Use a system prompt to set role and stable behavior
Put durable, cross-request behavior in the system prompt: role, expertise, tone, output conventions, hard constraints, and refusal policy. Put the specific, per-request task in the user turn. A sharp role ("You are a senior copy editor enforcing the AP style guide") raises the relevance and register of everything downstream. Keep the system prompt to what's genuinely stable across requests — task-specific detail belongs in the user message. (Keeping it stable also makes it cacheable — see caching below.)

### 8. Control the output format — and steer its opening
Never leave the shape of the output to chance when something downstream parses it.

- **Specify the exact format** — JSON schema, a Markdown template, a fixed set of headings, a length bound — and *show* a filled example rather than only describing it.
- **Use native structured outputs / JSON-schema modes when available.** Anthropic, OpenAI, and Google all now offer constrained decoding that *guarantees* schema-valid output — far more reliable than asking politely for JSON in prose (which yields valid-but-off-schema results). Prefer these over hand-rolled parsing. Note the tradeoffs: they can add first-call latency (grammar compilation) and may not support every JSON-Schema feature (recursion, numeric/length constraints); validate values in your app regardless.
- **Say what to do with missing fields** — null vs omit vs error — so the shape stays stable when the data isn't.
- **Steer the opening to kill preamble.** State it positively ("Begin directly with the summary; no preamble") and/or wrap the body in a format tag (`<summary>…</summary>`). *Legacy note:* prefilling the assistant turn (e.g. starting the response with `{`) was the classic way to force this, but **prefilling the last assistant turn is unsupported on current Claude models (4.6+) and returns a 400 error** — use structured outputs, an explicit no-preamble instruction, tool calling, or output tags instead.

### 9. Chain and decompose complex prompts
When one prompt is trying to do too much, split it. A prompt juggling five subtasks does each worse than five focused prompts each nailing one. Chain them: the output of one becomes the input to the next (extract → analyze → format), or run a generate-then-critique pass (draft → a separate prompt reviews it against a rubric → revise). The most common and useful chain is exactly that self-correction loop. Decomposition also makes each step testable in isolation. (When the chain needs runtime branching or tools, you've crossed into agent territory — see the next section and `agent-orchestration`.)

### 10. Build in guardrails and uncertainty handling
Tell the model what to do when it *can't* do the task well — this is where under-specified prompts leak or hallucinate.

- **Ground it:** "Answer only from the provided `<context>`. If the answer isn't there, say so — do not use outside knowledge."
- **Give it an out:** "If the request is outside your scope (billing, legal), say you can't help and point to <the right channel>." A model with no sanctioned way to decline will invent an answer.
- **Ask for calibrated uncertainty:** "If you're unsure, say so and state what would resolve it," rather than confident guessing.

## Match the prompt to the execution mode

The same task is prompted differently depending on whether it runs as a single model call, a tool-using agent, or a piece of a multi-agent system. Choosing the mode is `agent-orchestration`'s job; here is how the *prompt* changes at each level. The pattern as you move up: **you write less about the specific answer and more about the process, the boundaries, and how to decide what to do next.**

| | **Direct call** | **Single agent (tools)** | **Multi-agent** |
| --- | --- | --- | --- |
| Model decides "what next"? | No — one pass | Yes — loops over tools | Yes, at multiple levels |
| Prompt centers on | The exact output | Process + when to stop | Delegation + boundaries |
| Highest-leverage lever | Examples, output format | Tool descriptions, stop criteria | Per-worker task specs |
| Biggest failure mode | Vague spec | Never stops / stops too early | Duplicated or uncovered work |
| Prompt surfaces | One prompt | System prompt **+ tool schemas** | Orchestrator prompt + worker prompts + tool schemas |

### Prompting a direct call (one shot, no tools, no loop)
The model transforms input to output in a single pass; your code assembles everything first (including any RAG context). Because there's no loop to recover from a bad step, **the prompt must be complete up front.** Lean hardest on the direct-call levers:

- Nail the **output format** — schema, template, length — and enforce it with native structured outputs.
- Use **few-shot examples** to lock format and handle edge cases; this is where they pay off most.
- **Delimit input data** clearly and put it last.
- Specify **missing/edge-case behavior** explicitly — there's no second turn to fix it.
- Keep it self-contained: everything the model needs is in this one prompt.
- For deterministic transforms (classification, extraction), lower the temperature and constrain the output (enum-typed schema); even at temperature 0, outputs aren't guaranteed bit-identical, so constrained decoding is the real lever for structural determinism.

### Prompting a single agent with tools
Now the model runs in a loop — reasoning, calling tools, reading results, deciding what to do next. The prompt is a **policy for behavior over many turns**, not a spec for one answer. What changes:

- **Tool descriptions are prompt engineering — and the single highest-leverage surface.** "Providing extremely detailed [tool] descriptions is by far the most important factor in tool performance." Each tool's name, description, and parameter docs must be as clear as your main instructions: what it does, when to use it (and when *not* to), what each argument means, what it returns, and what it does *not* return. Aim for at least 3–4 sentences per tool, more if complex, with an example call for anything non-obvious. Bad tool docs send agents down completely wrong paths — more agent failures trace to weak tool descriptions than to a weak system prompt. (The tool schemas literally become part of the system prompt at request time.)
- **Design the agent-computer interface (ACI), not just the prompt.** Put yourself in the model's shoes: is it obvious how to use this tool? Prefer a few well-chosen high-impact tools over many overlapping ones; consolidate related operations; return semantically meaningful results (human-readable names, not bare UUIDs); "poka-yoke" arguments so mistakes are hard to make. Prune the toolset before you tune the prompt — too many tools is its own failure mode.
- **Set persistence and stop criteria explicitly.** Tell the agent to keep going until the task is fully resolved rather than stopping at the first partial result ("keep going until the user's query is completely resolved before yielding back") — *and* define "done" so it doesn't loop forever. State when to stop, when to ask the user, and when to give up and report.
- **Tell it to use tools instead of guessing:** "If you're unsure about file contents or structure, use your tools to read and gather the information — do not guess or make it up." But don't force tool calls unconditionally; if information is genuinely insufficient, it should ask the user rather than hallucinate inputs.
- **Instruct planning before acting** for multi-step tasks: "Plan before each tool call and reflect on the result of the previous one before the next." A short plan-then-act rhythm curbs flailing (this plus persistence and don't-guess measurably lifted agentic benchmark scores).
- **Handle tool failure and set an iteration budget:** what to do on an error or empty result (retry / try another tool / report), and a fallback ("if unsolved in N steps, summarize what you found and what's blocking you").

### Prompting a multi-agent system
Multiple agents coordinate, so prompts split by role. Since each agent is steered only by its prompt, prompt engineering is your primary lever — and the orchestrator/lead and the workers get *different* prompts with different jobs.

**Orchestrator / lead prompt** — teaches delegation, not doing:
- **Write detailed task specs for each worker.** The dominant multi-agent failure is vague delegation: workers duplicate each other, leave gaps, or misread scope. (Anthropic's example: a vague "research the semiconductor shortage" sent one subagent chasing the 2021 auto-chip crisis while others chased 2025 supply chains.) Every subtask handed down must state the **objective**, the **output format** expected back, **which tools/sources** to use, and **clear boundaries** (what's in and out of scope). "Find 2024 revenue for these 5 named competitors; return a table with source URLs" — not "research the market."
- **Scale effort to complexity, and cap it.** Tell the lead to size the work to the query: a simple fact-find warrants ~1 subagent and a handful of tool calls; a direct comparison ~2–4 subagents; only genuinely complex research justifies more, with clearly divided responsibilities. Left unguided, orchestrators over-decompose and (in the extreme) spawn dozens of agents for a trivial query. Put explicit spawn-count guidance and guardrails in the prompt.
- **How to synthesize.** Specify how to merge worker outputs, resolve conflicts between them, and what the final combined deliverable looks like.

**Worker / subagent prompt** — a focused specialist that can't see the whole picture:
- **Self-contained and narrow.** The worker sees only what the lead passed it, not the full conversation. Its prompt must restate the objective, the output contract, and its scope — assume no shared context.
- **Own tools, own stop criteria** — same single-agent discipline (tool docs, persistence, iteration cap), scoped to its one job, with least-privilege tool access.
- **Return in a fixed, compact shape** the lead can consume mechanically — structured results, not verbose narration. The lead's context fills fast; summarize on the way back up.

**Two hazards to design against — and a live disagreement:** (1) Cost. Single agents already use several times the tokens of a bare chat, and multi-agent systems roughly **15×** — plus latency and new coordination failure modes. (2) Context fragmentation. Anthropic's orchestrator-worker approach accepts that parallel workers each hold their own context (compression by parallelism); Cognition's "Don't build multi-agents" argues the opposite — that **workers acting on unshared context make conflicting implicit decisions** and produce incoherent results, so you should share full agent traces and often keep to a single continuous context. Both are right in their domain: parallel *read-mostly* work (research, review) tolerates isolation well; tightly *coupled* work (most coding) does not. Prompting can't rescue a decomposition that shouldn't have been split — exhaust a well-prompted single agent first, and see `agent-orchestration` for when the split is warranted.

## Iterate empirically — prompts are code

You cannot eyeball a prompt to "good." Treat prompting as an experimental loop against real inputs. (Anthropic names the prerequisites plainly: clear success criteria, a way to test against them empirically, and a first-draft prompt — *before* you optimize.)

- **Build an eval set first.** Representative and adversarial inputs with expected outputs or acceptance criteria. Make criteria specific and measurable — "under 0.1% of 10,000 outputs flagged by the content filter," not "safe outputs." Prioritize volume of cases over hand-graded perfection.
- **Baseline, then change one thing at a time.** Measure before and after each edit so you can attribute the change. Track accuracy, format-validity/success rate, consistency across repeated runs, and token/latency cost.
- **Pick the cheapest grader that works.** Code-based/exact-match is fastest and most reliable — use it wherever the output is checkable. Reserve LLM-as-judge for open-ended outputs (summaries, tone): give the judge a concrete rubric, have it reason first then emit a discrete verdict (`<thinking>`…`</thinking>` then `correct`/`incorrect` in `<result>`), and prefer a *different* model to judge than the one that generated. Avoid slow, expensive human grading where you can.
- **Analyze failures by category** — format error, hallucination, missed edge case, wrong scope — and let the category dictate the fix (format error → add a format example; hallucination → add a grounding instruction; ambiguity → tighten wording).
- **Use meta-prompting to bootstrap and refine.** A capable model is a good prompt engineer: ask it to draft a prompt (beats the blank page), or hand it a prompt plus a failure mode and ask it to diagnose and improve. Anthropic's Console ships a prompt generator and a prompt improver that do exactly this. Then re-run your evals.
- **Version prompts like code.** Store them in source control with the eval results that justified each change, so you can diff and roll back. Prompts drift; treat a prompt change like a code change.
- **Mind prompt caching.** Providers cache a stable *prefix*; on Anthropic the order is `tools` → `system` → `messages`, and a change at any level invalidates it and everything after. Keep static content (tools, system prompt, long reference docs, examples) fixed and at the front, and put the variable input at the end. This cuts cost and latency and rewards a clean, stable structure — another reason to organize static-first, variable-last. (Note: switching structured-output schemas injects a system prompt that can bust the cache.)

## Common pitfalls

- **Under-specification (the big one).** Leaving format, length, audience, or edge-case behavior implicit. The model fills the gap differently every run. Fix: say it.
- **Vague quality words.** "Good," "concise but thorough," "professional," "detailed" mean nothing operational. Replace with the concrete behavior you'd accept.
- **Negation-heavy prompts.** A wall of "don'ts" with no "dos." State the desired behavior positively; a bare prohibition can even cue the behavior it forbids.
- **Over-constraining / conflicting rules.** Piling on rules until they contradict, or restating the same rule five ways. Symptoms: the model ignores some rules (attention dilution), parrots your phrasing, or produces rigid, templated output. Fix with the cut test — cut *dead weight*, never *specificity*.
- **Over-prompting / shouting.** All-caps, `MUST`/`CRITICAL`, threats, bribes, and "if in doubt, use X" hedges. On current models these *overtrigger* and degrade behavior. Use plain, specific phrasing.
- **Untriggerable rules.** Rules for inputs that never occur. Dead weight; cut them.
- **Example pollution.** Wrong, off-distribution, or accidentally-patterned examples that teach the wrong thing.
- **Lost in the middle.** Burying the key instruction or fact in the middle of a long context. Front-load it, move the query to the end, restate the critical instruction, and have the model quote relevant passages first.
- **Instructions and data undelimited.** The model treats your data as instructions (or vice versa), and injection gets easier. Tag your sections.
- **Prompting the wrong mode.** A single mega-prompt doing five things (should be chained); a full agent for a one-shot transform (should be a direct call); a multi-agent system where one well-prompted agent would do.
- **Weak tool descriptions in agent prompts.** Polishing the system prompt while the tool docs stay vague — the tools are where the agent actually decides what to do.
- **Vague delegation in multi-agent prompts.** Fuzzy subtask specs → workers duplicate work or leave gaps.
- **Relying on deprecated tactics.** E.g. prefilling the last assistant turn (400 on current Claude), or fixed extended-thinking token budgets — verify a technique still applies to your target model.
- **No evals.** Editing prompts by vibe with no measurement. You can't tell improvement from regression.

## Checklist

Before shipping a prompt:

- [ ] The task, audience, and purpose (the *why*) are all stated
- [ ] Output format is specified exactly and shown, not just described
- [ ] Length, tone, and structure are concrete (no "concise," "professional," "good")
- [ ] At least one example — ideally 3–5, including a tricky case — matches the desired output format
- [ ] Instructions are positive ("do X") rather than a pile of "don'ts"; no all-caps/`MUST` shouting
- [ ] Edge cases and missing-data behavior are specified
- [ ] There's a sanctioned way to say "I don't know" / decline / flag uncertainty
- [ ] Reasoning space is provided if the task needs analysis, and separated from the answer
- [ ] Distinct sections (instructions, data, examples) are delimited; long data is first, query last
- [ ] Every rule passes the cut test (a real input triggers it) — no dead weight, no contradictions
- [ ] The prompt matches its execution mode (direct / single-agent / multi-agent)
- [ ] For agents: tool descriptions are as sharp as the instructions (≥3–4 sentences, when-not-to-use, returns); persistence, don't-guess, and an iteration cap are set
- [ ] For multi-agent: each subtask spec has objective, output format, tools, and boundaries; effort/spawn count is capped
- [ ] Techniques used are valid for the target model (no deprecated prefill / thinking-budget assumptions)
- [ ] Stable content is front-loaded for prompt caching; variable input is at the end
- [ ] There's an eval set and a baseline measurement to iterate against
- [ ] Colleague test: someone with minimal context could follow this prompt without confusion

## Resources

Anthropic's per-technique prompt-engineering pages now consolidate into a single living reference; cite it rather than the old sub-page URLs.

- Anthropic — Prompting best practices (consolidated living reference): https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices
- Anthropic — Prompt engineering overview: https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview
- Anthropic — Console prompting tools (generator + improver): https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/prompting-tools
- Anthropic — Structured outputs: https://platform.claude.com/docs/en/build-with-claude/structured-outputs
- Anthropic — Tool use (implement tool use): https://platform.claude.com/docs/en/docs/build-with-claude/tool-use/implement-tool-use
- Anthropic — Prompt caching: https://platform.claude.com/docs/en/docs/build-with-claude/prompt-caching
- Anthropic — Develop tests / evals: https://platform.claude.com/docs/en/docs/test-and-evaluate/develop-tests
- Anthropic — Building effective agents: https://www.anthropic.com/engineering/building-effective-agents
- Anthropic — Writing tools for agents: https://www.anthropic.com/engineering/writing-tools-for-agents
- Anthropic — How we built our multi-agent research system: https://www.anthropic.com/engineering/built-multi-agent-research-system
- OpenAI — GPT-4.1 prompting guide (agentic prompting: persistence, planning, tool-calling): https://developers.openai.com/cookbook/examples/gpt4-1_prompting_guide
- OpenAI — Prompt engineering guide: https://developers.openai.com/api/docs/guides/prompt-engineering
- OpenAI — Structured outputs: https://developers.openai.com/api/docs/guides/structured-outputs
- Google — Gemini structured output: https://ai.google.dev/gemini-api/docs/structured-output
- Cognition — Don't build multi-agents (context-sharing counterpoint): https://cognition.com/blog/dont-build-multi-agents
- Liu et al. 2023 — Lost in the Middle: https://arxiv.org/abs/2307.03172
- Companion skill: `agent-orchestration` (how many agents, and how they coordinate)
