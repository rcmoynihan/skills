---
name: state-it-plainly
description: Restate your own previous response in plain, jargon-free language — the bottom line, the key decisions, and what actually matters — so a long or noisy answer becomes easy to understand at a glance. Manual invocation only.
argument-hint: "(optional) what to focus on — e.g. 'just the plan', 'even simpler'"
disable-model-invocation: true
---

# State It Plainly

Your previous response was long, dense, or hard to parse, and the user wants to *actually understand
it*. Re-express **your own immediately-preceding response** in plain language and surface the parts
worth remembering. Explain it the way you would to a smart person who is new to this topic and short
on time — no assumed jargon, no wading required.

This is a **distillation of what you already said**, not a do-over. Do not research, read files, call
tools, or spawn agents. Do not introduce new claims, options, or work. Everything in the summary must
already be present in the response you are restating.

## What to summarize

The single assistant turn immediately before this invocation — the thing the user just read and found
hard to digest. If that response relayed a subagent's or tool's output, summarize the substance the
user was given, not the mechanics of how it arrived.

## How to write it

- **Translate, don't just shorten.** Rewrite in plain words. Expand or drop every acronym, codename,
  and piece of insider shorthand on first use. If a sentence needs domain knowledge to parse, rephrase
  it so it doesn't.
- **Lead with the punchline.** The most important takeaway goes first, in one or two sentences. The
  reader should be able to stop there and still have the gist.
- **Stay concrete.** Plain does not mean vague. Keep the load-bearing specifics — the actual
  recommendation, the real numbers, the specific files or names, the chosen option. Cut the noise
  around them, not the substance.
- **Strip the packaging.** Drop hedging, throat-clearing, process narration ("I explored several
  approaches…"), restated context the user already has, and anything that doesn't change what they'd
  do next. State conclusions directly.
- **Be honest to the original.** Don't sand off uncertainty or caveats that genuinely matter — if the
  response said something was risky or unknown, say so plainly too. Don't upgrade a tentative
  suggestion into a firm recommendation.

## Output format

Always open with **Bottom line** and close with **Worth questioning**. Between them, include only the
sections that carry real content *for this particular response* — pick from the menu below, title each
to fit, and skip the rest. Never emit an empty or padded section.

```
**Bottom line:** <the single most important takeaway, in 1–2 plain sentences>

## <a fitting header — e.g. The plan / The answer / What it found>
- <the substance, as a few short bullets a busy reader can scan>

## Key decisions            ← only if the response made or proposed choices
- <the choice> — <why, in plain terms>

## Numbers / specifics that matter   ← only if there are load-bearing figures, names, or files
- <the specific, and why it matters>

## Tradeoffs / caveats       ← only if the response weighed options or flagged real risks
- <the honest catch>

**Worth questioning:** <the one assumption or decision the whole thing hinges on — the thing you'd sanity-check before trusting it>
```

Keep it tight — usually well under a screen. If the original response was already short and clear,
say so in a sentence and give a one-line bottom line rather than manufacturing sections.

If the user passed arguments, treat them as focus or emphasis for the summary (e.g. "just the plan"
narrows it to the plan; "even simpler" means go plainer and shorter).

## Edge cases

- **The last response was itself a question to the user** (not an answer): restate, in plain terms,
  *what you're asking and why it matters* — and, if you laid out options, the practical difference
  between them — so the user can decide quickly. Skip "Worth questioning."
- **The last response was already a plain summary** (e.g. this skill just ran): say that briefly
  instead of re-summarizing the summary.
- **The response was mostly code or a raw artifact:** summarize what it does and the few things the
  user needs to know to use or judge it, rather than restating the code line by line.

## Example

*Restating a long response that compared three caching strategies and landed on one:*

```
**Bottom line:** Use Redis with a 5-minute expiry on the product-listing endpoint. It's the simplest
option that fixes the slow page, and we can tighten it later if data feels stale.

## The plan
- Cache the product-listing API response in Redis (an in-memory store — fast, but data lives outside
  the main database).
- Entries expire after 5 minutes, so listings can be at most 5 minutes out of date.
- Nothing else changes; the database stays the source of truth.

## Key decisions
- Chose Redis over caching inside the app's own memory — because we run several server copies, and
  an in-app cache would give each copy different data.
- Chose a fixed 5-minute expiry over clearing the cache on every edit — far simpler to build, at the
  cost of listings lagging up to 5 minutes.

**Worth questioning:** The whole thing assumes a 5-minute lag is acceptable to users. If listings
must update instantly after an edit, this approach is wrong and we'd need cache invalidation instead.
```
