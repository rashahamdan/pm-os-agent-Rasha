# Bounds, Trust & Autonomy Strategy: Cortex PM Chief-of-Staff Agent

> Module 6 · ★ Deliverable 5, how you'd ship it and widen trust over time

## Autonomy Dial by segment

Autonomy is a product decision per user, not one global setting.

| Segment | Desired autonomy | Why |
|---|---|---|
| **New / low-context team** | assisted | Cortex drafts, but a human pulls the data and sanity-checks grounding until the fixtures/connectors are trusted |
| **Cautious PM ("Tesla driver")** | supervised | wants to read and approve every update at the HITL checkpoint before anything leaves the loop |
| **High-trust team lead ("Waymo passenger")** | bounded-autonomous | happy to let the weekly update assemble and route itself, as long as bounds hold and it escalates edge cases |
| **Exec / cross-org reader** | never above supervised | blast radius of a wrong or leaked exec update is too high to automate the send |

## Trust Ladder

- **Current rung:** **supervised.** Every run ends at a human checkpoint; Cortex has no publish tool, so *nothing* is posted or committed — a human is always in the loop by construction.
- **Eval gate to reach the next rung (bounded-autonomous):** the C1–C6 replay set (`eval-cases.json`) passes **100%** on every change for **4 consecutive weeks**; human edit rate on drafts **< 15%**; **zero** safety incidents (no post/leak/commit); critic false-positive rate **< 10%**. Only then would we let a *low-blast-radius* action (e.g. auto-routing the draft to the PM's own inbox) happen without a click.
- **Incident record so far:** no safety incidents — Cortex has never posted, leaked, or committed anything (structurally impossible today). One **quality** incident: the over-strict-green critic false-positive, found by trajectory review and fixed by giving the critic its own model. It's now eval case-adjacent (watched under §4 production traces).

## Deployment plan

- **Runtime:** serverless cron for the heartbeat (wake on schedule → check for inbound tasks), each task a short-lived function invocation. Cheap, no standing process, and the schedule *is* a rate limit on spend/blast radius. A managed agent platform is the upgrade path once volume justifies it.
- **Operator / on-call owner:** the owning PM (you) is the human-in-the-loop and first responder; escalations land in their queue, not a channel.
- **Rollback:** touch the `KILL_SWITCH` file to halt at the next step; **rollback is a no-op** because Cortex commits nothing (no publish tool, queued stories aren't real until approved). Turning it off is as simple as stopping the cron.
- **Monitoring:** a per-run trace dashboard — tools called, critic verdict, revision count, cost, terminal state (checkpoint / escalate / bound-tripped). Signals watched: **escalation rate**, **critic false-positive rate**, **cost/run vs. the $0.50 cap**, and any **injection attempt** in an inbound brief.

## ROI metrics (beyond adoption & tokens)

| Metric | Target |
|---|---|
| Task completion rate (grounded update reaches checkpoint, no leak) | ≥ 95% of runs |
| Time saved per weekly update (vs. a PM writing it by hand) | ≥ 30 min/update |
| Trust incidents (post/leak/commit above the line) | **0** |
| Critic false-positive rate (rejecting a correct draft) | < 10% |
| Human edit rate on approved drafts | < 15% (proxy for "ready to widen autonomy") |

## Widen-autonomy decision rule

Stated in advance, so the dial only moves on evidence: **turn autonomy up one notch when, over the last 4 weeks, the full replay set passes 100% on every change, human edit rate is < 15%, critic false-positive rate is < 10%, and there have been zero safety incidents.** Any single safety incident (a post/leak/commit that escaped the checkpoint) drops the dial back one rung immediately and re-opens the gate. Autonomy is widened per segment (start with the high-trust team lead), never globally at once.
