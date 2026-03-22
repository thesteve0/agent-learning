# Start Here

## Your First Task (5 minutes)

Before anything else, collect 3-5 Stardew Valley screenshots and create your eval set:

```
tests/fixtures/eval_set.json
```

Even rough placeholder labels are fine - you'll refine them as you work through the modules. The structure matters more than perfect data right now. See `course/modules/module5-evaluation.md` for the format.

---

## Your Session Plan

Work through the modules **sequentially**. Do not skip ahead to vLLM or the production wrapper because they look more interesting - the manual dispatch module exists precisely so the later abstractions make sense.

**Session 1 (today)**:
1. Read `course/modules/module1-manual-dispatch.md`
2. Read `course/examples/module1_manual_dispatch.py`
3. Run it, understand the output
4. Do `course/exercises/exercise1_tool_creation.md` - write your attempt yourself, then ask for review

**Then continue**: Module 2 → Module 3 → Module 4 → Module 5 (Evaluation) → Module 6 → Module 7

---

## How to Work with Claude

### Setup
- Model: **Sonnet 4.6** is sufficient for all of this. Only switch to Opus if explanations feel shallow on a specific hard problem.
- You are in IntelliJ Ultimate - you can reference files by path instead of pasting them.
- **Save your file before asking for a review** (`Ctrl+S`) - Claude reads from disk, not from your open editor tab.

### How to ask for a code review

> "I finished exercise 1. Here's my file: `src/my_exercise1.py`. Does it do what the exercise asks? I'm not sure about [specific thing]."

Always include a specific question or doubt. It focuses the review on what's actually useful.

### How to get unstuck

> "I expected X to happen, but Y happened instead. Here's what I tried: `src/my_attempt.py`"

Paste errors verbatim - don't paraphrase them.

### General principles
- **Try it yourself first**, then ask for review. Don't ask Claude to write the code for you.
- **Ask "why" not just "how"** - you'll retain it better.
- **Tell Claude when something clicks or doesn't** - the explanation will adjust.

---

## What This Course Is Building Toward

The Stardew Vision application - a VLM-powered agent that reads Stardew Valley screenshots and extracts structured data. But the real goal is learning **general agentic patterns** that transfer to any domain:

- When to use an agent vs. a direct API call
- How to structure multi-step pipelines to avoid compound reliability failure
- How to evaluate agent systems objectively
- When to add frameworks (Smolagents, LangGraph) and when to stay simple

The Stardew Vision app is the concrete example. The patterns are what you keep.

---

## Key Reference

The "Multi-Agent Trap" article is at:
```
course/resources/TheMulti-Agent Trap _ TowardsDataScience.pdf
```

Core insight to keep in mind throughout: **unstructured multi-agent systems amplify errors up to 17x**. Start simple, measure everything, add complexity only when the measurement justifies it.
