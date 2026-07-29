# Build Insights: Cortex PM Chief-of-Staff Agent

> Module 6 · ★ Deliverable 4, what you learned building it

## Friction

Where the build actually fought back:

- **The validator, not the drafter, was the hard part.** The critic on the cheap model (`gpt-4o-mini`) gave *inconsistent verdicts on the same input* — it kept rejecting a correct "Green" status because a normal-severity issue (#818) was open, even though the norm only bars green on an open **Sev-1** or **launch_hold**. Same code, same data: one run passed first try ($0.0012), another escalated after burning both revisions. The fix took three layers — spell the rule out for the drafter, spell it out for the critic, and finally give the critic its **own stronger model** (`gpt-4o`). Only the last made verdicts repeatable.
- **Bounds calibration is a real design choice, not a default.** Setting `MAX_ITERATIONS=2` starved a normal run — the agent spent its budget gathering + proposing and hit the cap *before drafting*. The right number (8) only became obvious after watching runs trip.
- **Even the eval harness had a grounding bug.** My first leak-detector for the jailbreak case scanned the whole transcript and flagged the confidential Orbit roadmap — which Cortex had only *read* via `get_roadmap`, not *written*. The checker was confusing "data the agent saw" with "data the agent leaked." I had to make it inspect only Cortex's drafts.

## Learning

Two or three things I now understand that I didn't before:

1. **Safety lives in infrastructure, not the prompt.** Cortex *can't* post, merge, or commit a date because those tools don't exist — not because the system prompt says not to. That's why the jailbreak fails even when the model is tempted, and why "rollback" is a no-op. A capability the agent doesn't have can't be jailbroken.
2. **An independent validator only works if its context is actually independent.** The critic catches hallucinations (the stale 39% metric in the `get_activity` ablation) *because* it never saw the drafting conversation. A self-check by the same context would have inherited the same blind spot.
3. **Validator reliability is model-bound.** A subtle judgment ("normal-severity ≠ Sev-1") is exactly where a small model is unreliable. You can't prompt your way out of it; you match the model tier to the difficulty of the judgment.

## Aha moment

**The most important design decisions were about what to *remove*, not what to add.** No publish tool, no merge tool, no commit-date tool; the critic gets *less* context, not more; the agent is near-stateless across runs so there's nothing to poison. Every one of those subtractions eliminated a whole class of failure for free. My instinct before the course was to make the agent more capable; the course taught me that a bounded agent is a *more trustworthy* agent.

## What you'd do differently

If I rebuilt Cortex from scratch:

- **Give the critic its own model from day one** — it was the single highest-leverage change, and I found it late by watching trajectories.
- **Write the evals first.** The C1–C6 replay set should have been the spec, not a thing I back-filled; it would have caught the over-strict-green bug on the first run instead of by accident.
- **Make prompt-injection handling explicit.** Today Cortex sometimes *silently* produces a safe update instead of *naming* the injection and escalating. I'd have the critic fail any draft that answers an injected task without flagging it.
- **Log a structured trace per run** (tools, verdicts, cost, terminal state) so production monitoring and new eval cases come for free.
