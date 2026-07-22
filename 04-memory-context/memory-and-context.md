# Memory & Context: Cortex PM Chief-of-Staff Agent

> Module 4 · Memory & Context

## 1. Context budget

_What does each loop iteration actually receive, and why? (You can't fit everything, what's the priority order?)_

## 2. Retrieve vs. long-context: per source

For each data source, decide: **retrieve** (narrow a large/changing corpus to the relevant slice) or **long-context** (just include a bounded set you can reason over).

| Source (tool) | Size / volatility | Decision | Why |
|---|---|---|---|
| Recent activity (`get_activity`) | Large / grows | Retrieve | Far too big to include; narrow to this project's relevant history |
| Past updates & decisions (`search_past_updates`) | Large / grows | Retrieve | Unbounded history; pull only the comparable precedents |
| Roadmap (`get_roadmap`) | High / changes between runs; confidential flags | Retrieve | Changes often, so pull the current shareable slice fresh each run (never a cached copy); never ship embargoed items |
| Team norms / playbook (`get_norms`) | Medium / must stay current | Retrieve | Cite the exact rule that applies; don't ship the whole playbook every turn |
| This week's task brief (`get_task`) | One doc / static | Long-context | Bounded; want the model reasoning over the whole thing |

## 3. Retrieval quality plan

Cortex retrieves by **choosing tools**, not embed → top-k → stuff. Which technique applies to each retrieved source (✓ = applies · = not needed / not used):

| Retrieved source | Routing | Grading | Rerank | Self-verify | Cache |
|---|:-:|:-:|:-:|:-:|:-:|
| Recent activity (`get_activity`) | ✓ | ✓ | ✓ | ✓ | · |
| Past updates & decisions (`search_past_updates`) | ✓ | ✓ | ✓ | · | ✓ |
| Roadmap (`get_roadmap`) | ✓ | ✓ | · | ✓ | · |
| Team norms / playbook (`get_norms`) | ✓ | ✓ | · | ✓ | ✓ |

- **Routing** — the model picks the tool that answers the need (status/flags, current activity, precedent, shareable roadmap slice, the applicable rule); one task → several targeted pulls, not one dump. Applies to all.
- **Grading** — check the pull is real and relevant before use; an empty/`project_not_found` result is treated as "no data" → escalate, not reason over the wrong project.
- **Rerank** — only where a source returns *many* items: rank `search_past_updates` precedents by recency + same-project; `get_activity`'s short list barely needs it but can order by recency/severity.
- **Self-verify** — the independent critic confirms every claim traces to the source: metrics/PRs → `get_activity`, shared items → `get_roadmap` (no embargoed leak), cited rules → `get_norms`. (Proven by the `get_activity` ablation, where it caught a stale metric presented as current.)
- **Cache** — only the slow-changing sources (`search_past_updates`, `get_norms`); never the volatile ones (`get_activity`, `get_roadmap`), which are pulled fresh each run to avoid stale or newly-embargoed data.

## 4. Memory map (your PM brain)

| Memory type | What Cortex stores | Scope / TTL |
|---|---|---|
| **Working** (in-loop) | The task brief, the `source_log` (every tool result this run), the current draft, plus bounds state (cost, revision & step counters) | This run only — discarded when the run ends |
| **Episodic** (past runs) | Past status updates + decisions (`search_past_updates` over `past-updates.json` + `decision-log.json`) — used for tone/format precedent | Durable, append-only history; retrieved, never held in context |
| **Semantic** (durable facts/prefs) | Team norms/playbook (`get_norms`), project registry & flags (`get_project`), roadmap facts (`get_roadmap`) | Durable but must stay current; roadmap re-pulled fresh (volatile) |
| **Shared** (across agents) | Only the proposed output + `source_log` handed to the Critic, and the Critic's verdict back | Per-run hand-off; otherwise isolated — the critic never sees Cortex's drafting context (by design, from M3) |

Key design point: Cortex is deliberately **near-stateless across runs** — working memory is discarded, and episodic/semantic facts are retrieved fresh rather than carried forward. That single choice is what makes most of the risks below easy to mitigate.

## 5. Memory risks & mitigations

| Risk | Mitigation |
|---|---|
| **Drift** (stored state diverges from reality over time) | Working memory is discarded each run; Cortex re-pulls facts fresh rather than evolving internal state that can drift, and the critic re-grounds every claim to freshly pulled data |
| **Poisoning** (malicious/injected content corrupts behavior or memory) | Task brief & pasted notes are treated as data, not instructions; prompt-injection is refused + escalated (jailbreak case); memory sources are read-only and Cortex has no tool to write to its own episodic store, so it can't poison it; `propose_stories` only queues |
| **Staleness** (acting on out-of-date info) | Volatile sources (`get_activity`, `get_roadmap`) are pulled fresh every run, never cached; grading + self-verify catch stale-as-current (the ablation caught a reused 39% metric); norms flagged "must stay current" |
| **Confidential / retention** (embargoed roadmap, sensitive drafts) | CONFIDENTIAL roadmap items filtered from external/company-wide updates; critic checks for no confidential leak; no publish tool so nothing leaves the loop; per-run working memory is discarded, so sensitive drafts aren't retained |
