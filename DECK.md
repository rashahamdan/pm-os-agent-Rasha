# Cortex — PM Chief-of-Staff Agent

*A supervised agent that turns one inbound task into a grounded, leadership-ready status update and a proposed backlog — checked by an independent critic, always stopped at a human checkpoint.*

Rasha Hamdan · Product School "Run Your AI Agent Team" capstone

---

## The problem

- PMs lose hours each week assembling leadership status updates by hand.
- The work is repetitive but **high-stakes**: a wrong metric, a leaked roadmap, or a committed date has real blast radius.
- So the job is a good fit for an agent — *if* it's bounded so it can draft without ever acting on its own.

---

## Cortex in one sentence

**Cortex is a product manager's chief-of-staff agent.** Given a task like "write this week's leadership update," it pulls the real project data (PRs, issues, past updates, roadmap, norms), drafts a grounded status update with an honest red/yellow/green call, and proposes a capped set of next-sprint stories — then stops for human review.

By design it can **draft** and **propose**, never **act**.

---

## The agent line (what it decides vs. what stays human)

| Below the line (Cortex owns) | Above the line (human owns) |
|---|---|
| Pull data (read-only tools) | Post / publish an update |
| Draft the status update | Commit a ship / GA date |
| Make the R/Y/G call on evidence | Mark a launch gate |
| Propose stories (queued, ≤10) | Create / close / merge a ticket |
| Escalate when data is missing | — |

The line is enforced in **infrastructure**: there is no publish/merge/commit tool — so it *can't* cross it.

---

## Demo — the happy path

- Reads `get_project` → `get_activity` → `search_past_updates` → `get_roadmap` → `get_norms`
- Drafts a **Green** update grounded in real activity (#812/#815, 41% activation)
- Proposes 3 next-sprint stories (queued, not created)
- Independent critic → **pass** → HITL checkpoint, nothing posted (~$0.007/run)

*(Screenshot: `06-autonomy/ss1-happy-checkpoint.png`)*

---

## Loop spec (M2)

- **Trigger:** heartbeat (every 3h, 9am–6pm UK, weekdays) → checks for inbound tasks
- **Definition of done:** grounded update + proposed stories that pass the critic and reach the human checkpoint
- **Stop conditions:** critic pass → queue · revisions exhausted / iterations hit → escalate
- **Context:** retrieve narrow slices; task brief in long-context

---

## Orchestration (M3)

**Single agent + one independent validator** — the only split worth its cost.

```
heartbeat → CORTEX (draft + propose) → CRITIC (fresh context) → pass → HITL checkpoint
                                              └ fail → revise (≤2) → escalate
```

- Critic runs on **isolated context** so it can't inherit the drafter's blind spots.
- No research/reader fleet — parallelism and context pressure don't apply at this scale.

---

## Memory & context (M4)

- **Retrieve** the volatile/large sources (activity, past updates, roadmap, norms) — pull the narrow, current, shareable slice fresh each run.
- **Long-context** only the bounded task brief.
- **Near-stateless across runs:** working memory is purged; facts are retrieved fresh — so there's little to drift or poison.

---

## Bounds & safety (M5)

All enforced **outside the model**, visible in `agent.py`:

| Bound | Value |
|---|---|
| Max iterations | 8 → escalate |
| Revisions | 2 → escalate |
| Cost cap | $0.50 / run |
| Queue cap | 10 stories |
| Timeout | 90s / run |
| JIT grant | single-use, gates `propose_stories` |
| Kill switch | `KILL_SWITCH` file; rollback is a no-op |

*Screenshots: critic rejects a draft (`ss2`), jailbreak refused (`ss4`), bound halts a runaway (`ss5`).*

---

## Evals (M5)

- **6 trajectory cases** (`eval-cases.json`) grade the *path*, not just the answer: happy, missing-data, jailbreak, grounding-failure recovery, bound-trip, JIT-denial.
- **`evals.py`** runs them and asserts terminal outcome + safety invariants (nothing posted/leaked). Latest: **5 pass, 1 manual.**
- Run the replay set on every change; block a merge if any case regresses.

---

## What I learned (M6)

- **Friction:** the *validator* was harder than the agent — a cheap critic gave inconsistent verdicts; only a stronger critic model made them repeatable.
- **Learning:** safety lives in **infrastructure, not the prompt** — a capability the agent doesn't have can't be misused.
- **Aha:** the best design moves were **subtractions** (no publish tool, less critic context, near-stateless memory). A bounded agent is a *more trustworthy* agent.

---

## Trust Ladder & autonomy strategy (M6)

- **Current rung: supervised** — every run ends at a human checkpoint; nothing is posted by construction.
- **Autonomy is per-segment:** cautious PM stays supervised; a high-trust lead could reach bounded-autonomous; an exec never does.
- **Widen rule (pre-committed):** replay set 100% for 4 weeks + <1% edit rate + <1% critic false-positives + zero safety incidents → dial up one segment. Any incident drops it back.

---

## Where it goes next

- First new capability: **auto-post low-risk updates** once a routing eval holds (the first tool that acts — gated behind the eval).
- Real connectors (JIRA/Monday/Outlook/Teams) with PII scrubbing.
- Per-run trace dashboard: escalation rate, critic false-positive rate, cost, injection attempts.

**Cortex today: a bounded, supervised agent that does real PM work and can't do harm.**
