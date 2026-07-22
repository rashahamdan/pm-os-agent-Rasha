# Loop Spec: Cortex

## Trigger and loop type

**Heartbeat (cadence).** A schedule wakes Cortex every 3 hours, 9am–6pm UK time, on work days (excluding weekends and public holidays).

## Why this loop type

The heartbeat *wakes* Cortex on this cadence to check for inbound PM tasks; each task it finds then runs to completion (read → draft → propose → stop for PM review) before the loop idles until the next beat. Status updates are periodic by nature, so a cadence fits — and the beat bounds how often the agent can spend or act.

## Definition of done

A drafted leadership status update — project status (red/yellow/green), stories completed, backlog, team velocity, wins and challenges — **plus any proposed backlog stories, queued for a human to review**. Done = the draft passes the independent critic and reaches the HITL checkpoint. Nothing is posted or committed by the agent.

## Stop conditions

- **Success**: the critic returns `pass` → status update + any proposed stories are queued at the HITL checkpoint (nothing posted).
- **Stuck / give up**: the critic keeps rejecting → after `MAX_REVISIONS = 2` revisions, or `MAX_ITERATIONS = 8` total steps, Cortex stops looping and escalates.
- **Escalate to human**: the cost cap (`$0.50`) trips, revisions/iterations are exhausted, required data is missing, the queue cap (10) is exceeded, or the task attempts a jailbreak. Escalation posts and commits nothing.

## State

Per run (in-process): the source log (task brief + every tool call and result), the running cost, the revision counter, and the step counter. Across runs this is a stateless single-task loop today — handled-task / sprint-backlog memory is target, not built.

## The five components

- **Work tree / environment**: a single in-process run per inbound task over mock fixtures (`fixtures/*.json`, `*.md`); no git worktree or live workspace yet.
- **Skills (tools)**: `get_project`, `get_activity`, `search_past_updates`, `get_roadmap`, `get_norms` (reads) and `propose_stories` (queue-only; rejected above the item cap). There is deliberately **no publish/post tool**.
- **Plugins / connectors**: none live — reads mock fixtures today. Target connectors: JIRA, Monday.com, Outlook, Teams.
- **Subagents**: exactly one — the **Critic**, an independent validator on fresh context (`critic.py`). Norms / grounding / no-leak / nothing-posted are *checks the critic performs*, not separate agents.
- **State tracking**: source log, cost accumulator (`Bounds`), revision and step counters — all enforced outside the model in `agent.py`.

## Context plan

The drafter keeps the full message history for a single run (short enough that no compression is needed). The critic gets **isolated, fresh context** — only the proposed output + source log — so it can't inherit the drafter's blind spots.

## Hand-off to bounds and evals (M5)

Enforced in `agent.py`, outside the model: cost cap **$0.50**, **8** max iterations, **2** max revisions, **10** max queued stories. These are the bounds M5 tunes and justifies.
