---
title: "How I Think About Context Budgets When Building Agentic Systems"
description: "Context limits aren't just a technical constraint — they're an architectural concern. Here's the mental model I use to design agentic systems that stay sharp even at scale."
pubDate: 2026-03-18
---

You gave your agent a 1 million token context window. It still screws up.

Not because the model is bad. Not because your prompt is wrong. Because you're treating the context window like a bucket — just dump everything in and hope for the best.

The dirty secret: benchmarks consistently show better results *below* 200K tokens than at 1M. More context isn't better context. It's noisier context.

If you're building agentic systems seriously, you need to stop thinking about context size and start thinking about **context budget** — treating tokens like working memory you actively manage.

---

## The Mental Model: Context as Working Memory

Think about how you solve a hard problem. You don't try to hold your entire codebase, every conversation you've ever had, and your grocery list in your head at once. You pull in what's relevant to the current step, work with it, and move on.

LLMs work the same way — but they won't tell you when they're overwhelmed. The "lost in the middle" problem is real: attention degrades across long contexts. Critical information buried in the middle of a 500K token context will get less attention than the same information in a focused 50K context.

The question to ask before every agent step:

> *What does this agent need to know **right now** to do **this specific thing**?*

Everything else is noise. And noise costs you both tokens and quality.

```
❌ Bad prompt:
"Here is the entire codebase. Now fix the auth bug."

✅ Good prompt:
"Here is the auth module (auth.py, middleware.py).
The bug is on line 42 — JWT expiry isn't being checked.
Fix it without changing the public API."
```

The second prompt is cheaper, faster, and gets better results. Not because the model is different — because the context is better.

---

## The Explore → Act → Verify Loop

The most common mistake I see: agents diving straight into action without mapping the territory first.

The better pattern is three stages, each with a fresh context:

**1. Explore** — dispatch a subagent just to understand the codebase. It returns a *summary*, not raw file contents.

```
Explore subagent prompt:

Find the code that handles user authentication in this repo.
Return:
- File paths and line numbers
- How the auth flow works (2-3 sentences)
- Any related config or env vars

Do NOT return raw file contents. Summarize only.
```

The key constraint: "Do NOT return raw file contents." Without this, the subagent will dump everything into your root context and you're back to the bucket problem.

**2. Act** — the root agent works from the summary, not the raw codebase. Root context stays lean and focused.

**3. Verify** — another isolated subagent runs tests and reports back cleanly.

```
Verify subagent prompt:

Run the test suite. Report only:
- Pass/fail count
- Any failing test names and error messages

Suppress all other output.
```

Why suppress output? Because a test suite can produce thousands of lines. You don't want that in your root context — you want a signal, not noise.

The result: your root agent spends its tokens on *decisions*, not on reading files or parsing test output.

---

## Parallel vs. Serial — When to Split Work

The default agentic workflow is serial: do A, then B, then C. The problem is context accumulation — by the time you reach C, your context is carrying everything from A and B, most of which is now irrelevant.

Parallel subagents break this pattern. When tasks are independent, dispatch them simultaneously and merge the results.

```
Parallel dispatch prompt:

Use subagents to update all templates affected by this API change.
Templates are independent — run them in parallel.

Each subagent should:
1. Read its assigned template
2. Update the API call signature
3. Return only the diff
```

The decision rule is simple:

```python
# Use parallel subagents when:
# - tasks don't share state
# - order doesn't matter
# - you want speed + can use cheaper models for workers

# Use serial when:
# - step N needs output of step N-1
# - shared mutable state
# - the task requires a running mental model across steps
```

Bonus: parallel subagents can run on faster, cheaper models. Use Claude Haiku or Gemini Flash for workers, save your expensive model for the root decision layer.

---

## Specialist Agents — Cheap Workers for Expensive Problems

Your root agent is expensive. Its context is precious. Don't make it do grunt work.

Specialist subagents take on specific roles with custom system prompts. Three I use constantly:

**Code Reviewer** — fresh eyes, no history bias, brutally honest:

```
You are a senior code reviewer. You have no history of this codebase.
Review the diff provided. Flag:
- Bugs or edge cases
- Missing error handling  
- Security issues

Be blunt. No praise. Max 10 bullet points.
```

The "no history" instruction matters. You want the reviewer to approach the code cold, the way a new teammate would.

**Test Runner** — absorbs verbose output so root context doesn't have to:

```
Run the test suite for the auth module.
Report only:
- Total pass/fail count
- Stack traces for any failures
- Which test file each failure came from

Do not include passing test output.
```

**Debugger** — dedicated token budget for root cause analysis:

```
This function is returning None unexpectedly.
Here is the function, its tests, and the failing stack trace.

Reason through what could cause this.
Do NOT fix it yet — identify the root cause only.
```

The "don't fix it yet" constraint is deliberate. Debugging and fixing are separate cognitive tasks. A subagent that tries to do both often does neither well.

---

## Rules I Actually Follow

After building enough of these systems, I've landed on a few hard rules:

```
✅ Keep root context under 100K for reliable reasoning
✅ Never dump raw file contents — always summarize first  
✅ One subagent per concern, not per file
✅ Root context = decisions. Subagents = grunt work.
✅ When something goes wrong: check the context before the prompt
```

That last one is underrated. When an agent behaves unexpectedly, most people immediately start tweaking the prompt. But 80% of the time, the real problem is that the context contains contradictory information, stale state, or too much noise. Fix the context first.

---

## What This Looks Like in Real System Design

Here's how I'd design a multi-step code migration agent — say you're migrating a codebase from one API version to another:

```
root agent (decision layer, target: <80K tokens)
├── explorer subagent     → maps affected files, returns summary
├── planner subagent      → returns ordered task list
├── worker-1 subagent     → migrates file A  (parallel)
├── worker-2 subagent     → migrates file B  (parallel)
├── worker-3 subagent     → migrates file C  (parallel)
└── reviewer subagent     → reviews merged diff, flags issues
```

Rough token budget:
- Root agent: 60–80K (summaries + decisions only)
- Explorer: 150K (can read a lot, returns ~2K summary)
- Workers: 30–50K each (focused on one file)
- Reviewer: 50K (receives final diff only)

The root agent never sees a raw file. It sees summaries, task lists, and diffs. Its entire job is to orchestrate and decide.

This is the mindset shift that matters: **you're not prompting one agent, you're architecting a team.**

---

## Where This Is Heading

Context windows aren't going to 10M tokens and solving this. The research is clear — attention doesn't scale linearly. Models are getting smarter about *using* context, not just having more of it.

The engineers who build great agentic systems in the next few years won't be the ones who write the best prompts. They'll be the ones who think in budgets, design in layers, and treat context as the scarce resource it is.

---

*For a deeper dive into the patterns referenced here, Simon Willison's [Agentic Engineering Patterns](https://simonwillison.net/guides/agentic-engineering-patterns/) guide is the best reference I've found.*
