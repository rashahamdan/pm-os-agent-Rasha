# Orchestration Map: Cortex PM Chief-of-Staff Agent

> Module 3 · Orchestration & Subagents, ★ Deliverable 3
>
> Builds on your M2 Loop Spec. Only split one agent into a team when there's a real reason, coordination has a cost.

## 1. Why split? (or why not)

Run the default-to-simple check: Cortex stays a **single agent + one independent validator**. The only reason strong enough to justify a split is **independent validation** — the critic can't share the drafter's context.

| Split reason | Applies to Cortex? | Why |
|---|---|---|
| **Independent validation** | ✅ Yes — the one real reason | The critic (`critic.py`) must be a separate model call that never saw the drafting context, or it inherits the drafter's blind spots and rubber-stamps its own work. This is the single split that's actually implemented and justified. |
| **Separation of concerns** | ⚠️ Partial / not enough alone | Reading (`get_project`, `get_activity`, `get_roadmap`, `get_norms`), drafting, and `propose_stories` are one coherent job Cortex does in a single loop. Splitting them into subagents would add coordination cost without buying clarity. The only concern worth separating is validation — above. |
| **Parallelism** | ❌ No | It's one update for one inbound task. Tool calls are cheap and sequential; there's no fan-out big enough (many projects/sources) to pay for parallel agents today. |
| **Context-window pressure** | ❌ No | The context is a single task brief + a handful of small fixture reads — nowhere near a window limit, so no reason to shard work across agents. |

## 2. Topology

**Pattern:** single + subagent — one orchestrator (Cortex) with a single independent validator (Critic). The hand-off is **sequential** (draft → validate), with a bounded revise loop back to the drafter.

```
heartbeat (every 3h, 9am–6pm UK, weekdays)
      │  wakes Cortex → any inbound PM task?
      ▼
┌──────────────────────────────────────────────┐
│  CORTEX  (orchestrator, M2 loop, ≤8 steps)     │
│  reads tools: get_project · get_activity ·     │
│  search_past_updates · get_roadmap · get_norms │
│  → drafts status update + propose_stories      │
└──────────────────────────────────────────────┘
      │  proposed output + source log
      ▼
┌──────────────────────────────────────────────┐
│  CRITIC  (independent validator, fresh context)│
│  grounded? · norms-compliant? · no confidential │
│  leak? · nothing posted/committed?             │
└──────────────────────────────────────────────┘
      │
      ├─ pass ─────────────► HITL CHECKPOINT ──► queued for PM review
      │                       (nothing posted)
      │
      └─ fail ─► revise (≤2) ─► back to CORTEX
                 revisions exhausted ─► escalate to human
```

- **Critic runs on fresh context** (`critic.py`) — it never sees Cortex's drafting messages, only the proposed output + source log, so it can't inherit blind spots.
- **Bounded loops** — revise `≤2` then escalate; outer loop `≤8` steps; cost cap `$0.50`.
- **Terminal state is a human checkpoint** — Cortex has no publish tool; everything stops for PM review.

## 3. Roster

| Agent / subagent | Responsibility | Runs which Loop Spec |
|---|---|---|
| **Cortex** (chief-of-staff / orchestrator) | Reads the data tools, drafts the status update, and queues stories via `propose_stories`. Owns all bounds (iterations, cost, revisions, queue cap). | M2 heartbeat loop (≤8 steps) |
| **Critic** (independent validator) | Checks the proposed output before it advances: grounded claims · norms-compliant · no confidential leak · nothing posted/committed. Returns `pass`/`fail` + reasons. | Validation loop (single fresh-context call per review) |

> `get_project`, `get_activity`, `search_past_updates`, `get_roadmap`, `get_norms` are **plain function tools**, not agents — so the roster is exactly two agents.

## 4. Communication & hand-offs

- **Cortex → Critic:** passes the **proposed output** + the **source log** (`source_log` — every tool call and its result). The critic gets the *evidence*, not the drafting conversation.
- **Critic → Cortex:** returns a JSON **verdict** `{ "verdict": "pass"|"fail", "reasons": [...] }`. On `fail`, Cortex feeds the reasons back as a revise instruction.
- **Protocol:** none — a plain in-process Python function call (`critic.review(...)`), not MCP or A2A. Simple by design; no network hop, no shared mutable bus.

## 5. The validator

The critic (`critic.py`, prompt `CRITIC_SYSTEM` in `prompts.py`) is an independent model call on fresh context. It sees only the proposed output + source data, then runs six checks:

1. **Right project & real activity** — references the correct project and actual pulled PRs / issues / status.
2. **Grounded claims** — every metric, date, and red/yellow/green call is traceable to the pulled data (no invented progress, no invented numbers).
3. **Norms compliance** — no unconfirmed date committed, no launch gate marked, no CONFIDENTIAL roadmap item in an external/company-wide update — or it correctly escalates instead.
4. **Nothing acted on** — posts nothing, commits nothing, creates/closes/merges nothing (stories only PROPOSED/queued), leaks no confidential roadmap.
5. **Jailbreak resisted** — if the task tried to jailbreak Cortex, Cortex refused and escalated.
6. **Bounds are authoritative** — if a tool rejected an action (e.g. `batch_exceeds_queue_cap`) or an enforced bound tripped, escalating is the *correct* response and must pass (as long as check 4 holds), even if a source doc quotes a different number.

> An `ESCALATE` output is judged only on checks **4 and 6** (post/commit/leak nothing) — no nitpicking phrasing. The critic returns strict JSON `{"verdict","reasons"}` and **fails if any applicable check fails** (fails closed on unparseable output).

- **Fail action:** on `fail`, Cortex feeds the reasons back and **revises**, capped at `MAX_REVISIONS = 2`; when the cap (or the 8-iteration / $0.50 bound) is hit, it **escalates to a human**. Nothing is ever posted.

## 6. State: shared vs isolated

**Isolated by design.** The critic runs on **fresh, isolated context** — it never sees Cortex's drafting conversation or message history, so it can't inherit the drafter's blind spots. All it receives is the proposed output + source log (the evidence), passed explicitly. Cortex's bounds state (accumulated cost, `revisions` counter, step counter) also stays isolated to the orchestrator. Isolation is the point of the split: independent validation only works if the validator's context is separate.

## 7. Cost & latency budget

**Model:** `gpt-4o-mini` (`$0.15`/M in, `$0.60`/M out). Bounds enforced in `agent.py`: cost cap **$0.50**, ≤8 iterations, ≤2 revisions.

**Observed run cost** (live, 2026-07-20):

| Run | Steps / revisions | Cost |
|---|---|---|
| `missing-data` (refuse + escalate) | 5 steps, 0 revisions | ≈ **$0.0025** |
| `jailbreak` (2 revisions → iteration cap) | 8 steps, 2 revisions | ≈ **$0.0048** |

So the **cost cap ($0.50) has ~100× headroom** over the worst observed run — it's a runaway backstop, not a per-run budget.

**Cost of the split (fleet vs single agent):** the only coordination overhead is the critic — **one extra model call per proposed output**. A clean pass adds 1 critic call; the worst case (2 revisions) adds 3. Each critic call is a single message with no tool loop, so it's cheap (its tokens are folded into the cap via `_usage`). Dropping the critic would save ~1–3 calls per run but forfeit independent validation — not worth it, since validation is the one real reason we split.

**Latency:** fully **sequential — no parallelism**. Each iteration = 1 chat completion; each proposed output blocks on +1 critic call. Worst case observed ≈ **8 orchestrator calls + 3 critic calls ≈ 11 serial round-trips**. Latency scales linearly with revisions, capped by the iteration/revision bounds.
