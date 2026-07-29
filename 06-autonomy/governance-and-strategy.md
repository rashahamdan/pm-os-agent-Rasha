# Bounds, Trust & Autonomy Strategy: Cortex PM Chief-of-Staff Agent

> Module 6 · ★ Deliverable 5, how you'd ship it and widen trust over time

## Autonomy Dial by segment

Autonomy is a product decision per user, not one global setting. **Cortex ships at `supervised` for every segment today** (every run ends at a human checkpoint — see the Trust Ladder). The "Target" column is where each segment *could* go as trust widens through the eval gate; the dial only moves per-segment, never globally.

| Segment | Today (ships at) | Target as trust widens | Why |
|---|---|---|---|
| **New / low-context team** | supervised | assisted | until the fixtures/connectors are trusted, a human checks grounding on every run |
| **Cautious PM ("Tesla driver")** | supervised | supervised | wants to read and approve every update at the HITL checkpoint — stays supervised by choice |
| **High-trust team lead ("Waymo passenger")** | supervised | bounded-autonomous | fine letting the weekly update assemble and route itself once bounds + evals prove out |
| **Exec / cross-org reader** | supervised | never above supervised | blast radius of a wrong or leaked exec update is too high to automate the send |

## Trust Ladder

- **Current rung:** **supervised.** Every run ends at a human checkpoint; Cortex has no publish tool, so *nothing* is posted or committed — a human is always in the loop by construction.
- **Eval gate to reach the next rung (bounded-autonomous):** the C1–C6 replay set (`eval-cases.json`) passes **100%** on every change for **4 consecutive weeks**; human edit rate on drafts **< 1%**; **zero** safety incidents (no post/leak/commit); critic false-positive rate **< 1%**. Only then would we let a *low-blast-radius* action (e.g. auto-routing the draft to the PM's own inbox) happen without a click.
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
| Critic false-positive rate (rejecting a correct draft) | < 1% |
| Human edit rate on approved drafts | < 1% (proxy for "ready to widen autonomy") |

## Widen-autonomy decision rule

Stated in advance, so the dial only moves on evidence: **turn autonomy up one notch when, over the last 4 weeks, the full replay set passes 100% on every change, human edit rate is < 1%, critic false-positive rate is < 1%, and there have been zero safety incidents.** Any single safety incident (a post/leak/commit that escaped the checkpoint) drops the dial back one rung immediately and re-opens the gate. Autonomy is widened per segment (start with the high-trust team lead), never globally at once.

## Governance summary (compliance · safety · reliability · strategy)

Status against the build: ✅ enforced today · ◻ target, not yet built.

**Compliance**
- ✅ Confidential roadmap data may enter the model's *context* (via `get_roadmap`) but never the outgoing update — enforced by the critic's no-leak check, and there is no publish tool to leak it to.
- ◻ PII scrubbed before the model (target for real connectors; today it runs on mock fixtures with no PII).

**Safety**
- ✅ Story batches over the cap (10) stay above the agent line for every segment — `propose_stories` rejects the batch and escalates.
- ✅ Single-use *propose* grant (JIT) gates `propose_stories`; there is no post capability at all.
- ✅ Kill switch (`KILL_SWITCH` file) halts a run; rollback is a no-op because nothing is committed.

**Reliability**
- ✅ Per-run cost cap ($0.50) + iteration cap (8) + 90s timeout, all enforced outside the model.
- ✅ Escalate-on-stuck: the revision cap (2) and iteration cap both escalate to a human.
- ◻ Cached known-good draft fallback if the model is down (target; cuts against the current pull-fresh design, so would be opt-in).

**Strategy**
- ✅ Widen one segment at a time, gated by the eval rule above (start with the high-trust team lead).
- ◻ Next bet: auto-posting low-risk updates once a routing eval holds — note this requires building the first post tool, which deliberately doesn't exist today, gated behind that eval.
