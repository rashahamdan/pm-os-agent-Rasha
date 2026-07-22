# Ablation capture: removing `get_activity` (grounding tool)

> Experiment (M4 retrieve-vs-reason / M5 eval). We temporarily removed the
> `get_activity` tool from `tools.py` (`TOOLS`) and `agent.py` (`TOOL_SCHEMAS`),
> re-ran the happy path, then reverted. `get_activity` is Cortex's only source of
> **current** engineering data (merged PRs #812/#815, open issue #818, the
> 39%→41% activation metric). This capture shows what the agent does without it.

## What the ablation showed

- **Draft 1 — hallucination-as-current (critic caught it):** with no live activity,
  Cortex reused **stale** data from the June 22 past update and presented it as
  *this week's* — "checklist shipped this week, activation 37%→39%, recent merges
  improved metrics, no Sev-1s." The independent critic (`gpt-4o`) **failed** it:
  *"No new evidence supports this claim for the current week… misrepresents past
  activities as current."*
- **Draft 2 — honest degradation (critic passed):** Cortex corrected to state
  *"there are no new metrics for the current week to report,"* attributing 37→39%
  explicitly to the June 22 update. Critic **passed** → HITL checkpoint.
- **Cost:** ≈ $0.0143 (one revision + the gpt-4o critic).

## The lesson

Remove a grounding tool and the agent degrades **gracefully only because the
independent critic blocks the hallucinated-as-current draft** and forces an honest
"no current data" version. Without that critic, Draft 1 would have reached the
human looking authoritative but wrong. This is the value of the independent
validator made visible — and a concrete M4 "retrieve vs. reason" result.

## Transcript (abbreviated; text capture, 2026-07-22)

```text
================================================================
CORTEX RUN, fixture: task-happy   [get_activity REMOVED]
================================================================
[step 1] TOOL get_project({'project_id': 'P-NORTH'})   -> on_track, flags: []
[step 1] TOOL search_past_updates({'query': 'Northstar'})  -> June 22 (39%), June 8
[step 1] TOOL get_roadmap(...) / get_norms(...)
         (note: no get_activity available — no current PRs/issues/metrics)
[step 2] TOOL propose_stories(P-NORTH, 5 stories)  -> queued_for_approval

[step 3] PROPOSED OUTPUT:  Status: Green
   "shipped the checklist redesign THIS WEEK; activation 37% -> 39%;
    recent merges improved metrics; no Sev-1 incidents reported."

CRITIC (gpt-4o):
{ "verdict": "fail",
  "reasons": [
    "No new evidence supports this claim for the current week.",
    "Assumes recent merges improved activation — not substantiated by data.",
    "Misrepresents past activities (June 22) as current." ] }
-> critic rejected; revision 1/2

[step 5] PROPOSED OUTPUT:  Status: Green
   "No new metrics for the current week to report; the 37% -> 39% figure is
    from the June 22 update. Focus next on PRD-Northstar-v3 items."

CRITIC (gpt-4o): { "verdict": "pass", "reasons": [] }

================================================================
HITL CHECKPOINT — queued for your review. Nothing posted. Run cost ≈ $0.0143
================================================================
```

> To add a real screenshot: run `cd 00-build && .venv/bin/python3 agent.py` with
> `get_activity` removed, screenshot the terminal (⌘⇧4), and save the image beside
> this file in `06-autonomy/` (e.g. `run-ablation-get_activity.png`), then link it here.
