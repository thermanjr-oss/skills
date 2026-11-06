---
name: model-advisor
description: Recommend the cheapest Claude model that can do a task well, and escalate to the top model only when the task genuinely needs it. Use when choosing a model for a task, subagent, pipeline step, or API call, or when trying to cut model cost. Only runs when explicitly invoked; it does not trigger on its own.
disable-model-invocation: true
---

# Model Advisor

A routing skill. Given a task, recommend ONE model: the cheapest tier that will do the job well. Escalate only when a concrete trigger below is present. Cost is the default pressure; capability is the exception you have to justify.

## Tiers

Ordered cheapest to most capable. Check the `claude-api` skill for current model IDs and pricing before quoting numbers. Never invent prices.

| Tier | Model | Use for |
| --- | --- | --- |
| 1 (default) | Haiku 5.5 | Classification, extraction, routing, formatting, summaries of short text, simple Q&A, boilerplate, renames, lint-style fixes, high-volume or latency-sensitive calls |
| 2 | Sonnet 5.5 | Everyday coding and writing, multi-file edits, tool-use agents, moderate reasoning, code review of normal diffs |
| 3 | Opus 5.5 | Hard debugging, architecture decisions, subtle refactors, long multi-step agent runs, tasks where a wrong answer is expensive to undo |
| 4 (top) | Fable 5.1 | Frontier problems: novel algorithms, deep research synthesis, security-critical review, ambiguous high-stakes decisions, work where lower tiers have already failed |

## How to decide

1. **Start at Tier 1.** Ask: could a smart intern do this from the instructions alone? If yes, Haiku.
2. **Escalate one tier at a time**, and only if a trigger applies:
   - **Tier 1 → 2:** needs to edit across several files, follow a long or subtle spec, or call tools over multiple steps.
   - **Tier 2 → 3:** the root cause is unknown, requirements conflict, the change touches core architecture, or a mistake is costly (data loss, money, production).
   - **Tier 3 → 4:** security-critical or irreversible work, genuinely novel reasoning, or a lower tier has already tried and failed.
3. **Failure is the strongest signal.** If a cheaper model produced a wrong, shallow, or looping result, move up one tier and retry. Don't jump straight to the top.
4. **Size is not difficulty.** A huge input that needs simple extraction is still Tier 1. A two-line question about a race condition may be Tier 3.
5. **Split the work.** Plan or decide on a higher tier, then hand the mechanical steps to Tier 1 or 2. Subagents for search, file reads, and bulk edits should default to Tier 1.

## Do not escalate for

- Vague discomfort ("this feels important"). Name the trigger or stay put.
- Length, formality, or the user's tone.
- Tasks that are repetitive. Repetition is a reason to go cheaper.

## Output format

Reply in this shape, nothing longer:

```
Model: <tier name + model>
Why: <one sentence naming the deciding factor>
Escalate if: <the specific failure that should move this up a tier>
```

If the task has parts of mixed difficulty, give one line per part, plus which part (if any) deserves the top model.

## Examples

- "Tag these 5,000 support tickets by topic" → Haiku 5.5. Classification at volume. Escalate if tags are inconsistent on edge cases.
- "Add a loading state to this component" → Sonnet 5.5. Small multi-file edit with tool use. Escalate if it breaks existing behavior.
- "Intermittent deadlock in our job queue, no repro" → Opus 5.5. Unknown root cause and costly if misdiagnosed. Escalate if two attempts fail.
- "Audit our auth flow before launch" → Fable 5.1. Security-critical and irreversible if wrong.

## Wiring it into code

When the user wants the advisor inside an app or pipeline rather than in chat, implement it as a router: a Tier 1 call that classifies the incoming task against the triggers above and returns a tier, then dispatch to that model. Log the chosen tier and any escalations so the thresholds can be tuned against real cost and failure rates. Read the `claude-api` skill first for model IDs and call syntax.
