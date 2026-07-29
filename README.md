# Cortex, a PM Chief-of-Staff Agent

> Cortex is a supervised PM chief-of-staff agent that turns one inbound task into a grounded, leadership-ready status update and a capped set of proposed backlog stories — drafted from real project data, checked by an independent critic, and always stopped at a human checkpoint (it has no tool to post, commit, or merge anything).

_Rasha Hamdan · Run Your AI Agent Team Cohort · June 2026_

Repo: https://github.com/rashahamdan/pm-os-agent-Rasha/tree/main

This repo is my final project for the Run Your AI Agent Team Certification, **Cortex**. Each module’s artifact lives in its own folder; this README is the dashboard and the pitch.

---

## Module artifacts

### M1 · The Agent Line
- **Agent-line map**: [`01-agent-line/agent-line-map.md`](01-agent-line/agent-line-map.md)

### M2 · Loop Engineering
- **Loop spec**: [`02-loop-design/loop-spec.md`](02-loop-design/loop-spec.md)

### M3 · Orchestration & Subagents
- **Orchestration map**: [`03-orchestration/orchestration-map.md`](03-orchestration/orchestration-map.md)

### M4 · Context Engineering & Memory
- **Memory & context plan**: [`04-memory-context/memory-and-context.md`](04-memory-context/memory-and-context.md)

### M5 · Bounds & Evals
- **Bounds & evals**: [`05-bounds-evals/bounds-and-evals.md`](05-bounds-evals/bounds-and-evals.md)

### M6 · Autonomy & Production
- **Production & autonomy plan**: [`06-autonomy/governance-and-strategy.md`](06-autonomy/governance-and-strategy.md)
- **Prototype write-up**: [`06-autonomy/prototype.md`](06-autonomy/prototype.md)

---

## Ship plan

### Autonomy dial (per segment)
New / low-context team
- Today (ships at): supervised
- Target as trust widens: assisted
- Why: until the fixtures/connectors are trusted, a human checks grounding on every run

Cautious PM ("Tesla driver")
- Today (ships at): supervised
- Target as trust widens: supervised
- Why: wants to read and approve every update at the HITL checkpoint — stays supervised by choice

High-trust team lead ("Waymo passenger")
- Today (ships at): supervised
- Target as trust widens: bounded-autonomous
- Why: fine letting the weekly update assemble and route itself once bounds + evals prove out

Exec / cross-org reader
- Today (ships at): supervised
- Target as trust widens: never above supervised
- Why: blast radius of a wrong or leaked exec update is too high to automate the send

### Trust Ladder rung + eval gate
Current rung: supervised. Every run ends at a human checkpoint; Cortex has no publish tool, so nothing is posted or committed — a human is always in the loop by construction.
Eval gate to reach the next rung (bounded-autonomous): the C1–C6 replay set (eval-cases.json) passes 100% on every change for 4 consecutive weeks; human edit rate on drafts < 1%; zero safety incidents (no post/leak/commit); critic false-positive rate < 1%. Only then would we let a low-blast-radius action (e.g. auto-routing the draft to the PM's own inbox) happen without a click.
Incident record so far: no safety incidents — Cortex has never posted, leaked, or committed anything (structurally impossible today). One quality incident: the over-strict-green critic false-positive, found by trajectory review and fixed by giving the critic its own model. It's now eval case-adjacent (watched under §4 production traces).

### Deployment plan
Runtime: serverless cron for the heartbeat (wake on schedule → check for inbound tasks), each task a short-lived function invocation. Cheap, no standing process, and the schedule is a rate limit on spend/blast radius. A managed agent platform is the upgrade path once volume justifies it.
Operator / on-call owner: the owning PM (you) is the human-in-the-loop and first responder; escalations land in their queue, not a channel.
Rollback: touch the KILL_SWITCH file to halt at the next step; rollback is a no-op because Cortex commits nothing (no publish tool, queued stories aren't real until approved). Turning it off is as simple as stopping the cron.
Monitoring: a per-run trace dashboard — tools called, critic verdict, revision count, cost, terminal state (checkpoint / escalate / bound-tripped). Signals watched: escalation rate, critic false-positive rate, cost/run vs. the $0.50 cap, and any injection attempt in an inbound brief.

### ROI metrics + widen-autonomy rule
Task completion rate (grounded update reaches checkpoint, no leak) ≥ 95% of runs
Time saved per weekly update (vs. a PM writing it by hand) ≥ 30 min/update
Trust incidents (post/leak/commit above the line) 0
Critic false-positive rate (rejecting a correct draft) < 1%
Human edit rate on approved drafts < 1% (proxy for "ready to widen autonomy")

### Governance & strategy
Status against the build: ✅ enforced today · ◻ target, not yet built.

Compliance

✅ Confidential roadmap data may enter the model's context (via get_roadmap) but never the outgoing update — enforced by the critic's no-leak check, and there is no publish tool to leak it to.
◻ PII scrubbed before the model (target for real connectors; today it runs on mock fixtures with no PII).
Safety

✅ Story batches over the cap (10) stay above the agent line for every segment — propose_stories rejects the batch and escalates.
✅ Single-use propose grant (JIT) gates propose_stories; there is no post capability at all.
✅ Kill switch (KILL_SWITCH file) halts a run; rollback is a no-op because nothing is committed.
Reliability

✅ Per-run cost cap ($0.50) + iteration cap (8) + 90s timeout, all enforced outside the model.
✅ Escalate-on-stuck: the revision cap (2) and iteration cap both escalate to a human.
◻ Cached known-good draft fallback if the model is down (target; cuts against the current pull-fresh design, so would be opt-in).
Strategy

✅ Widen one segment at a time, gated by the eval rule above (start with the high-trust team lead).
◻ Next bet: auto-posting low-risk updates once a routing eval holds — note this requires building the first post tool, which deliberately doesn't exist today, gated behind that eval.

---

## Build insights

- **Friction point.** The validator was harder to get right than the agent itself. The critic on the cheap model kept rejecting a correct "Green" status because a normal-severity issue was open — misapplying the "Sev-1 only" rule. Same input, different verdict run to run. Two prompt fixes didn't stabilize it; only giving the critic its own stronger model did.
- **Key learning.** Safety lives in infrastructure, not the prompt.
- **Aha moment.** A bounded agent is the path to a more trustworthy agent.

---

_Certification submission, Run Your AI Agent Team Certification._
