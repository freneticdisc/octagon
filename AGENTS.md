# Agent Rules of Engagement

## Architecture
- **Async-first:** all LLM I/O via `asyncio` — no blocking calls
- **State:** LangGraph `StateGraph` only — no global variables
- **LLM calls:** LiteLLM exclusively — no direct provider SDK imports
- **Data structures:** Pydantic v2 for all schemas (Config, State, EvictionRecord)

## Coding Standards
- Extend existing code; no refactoring without a written "Proposed Change Plan" approved first
- Mandatory type hints on all function signatures
- Gracefully handle API timeouts and local model connection failures — log and continue or evict, never crash the graph
- Any change to core logic or architecture requires human review before implementation

## Eviction Logic (`core/eviction.py`)

| Trigger | Method | Threshold |
|---|---|---|
| Topic deviation | `litellm.embedding` cosine similarity vs. original requirement | < 0.4 → evict |
| Repetition | cosine similarity vs. participant's last 2 responses | > 0.9 → evict |
| Token overrun | cumulative tokens vs. participant `token_limit` | exceeded → evict |

On eviction: **must** log participant name, violation type, exact score, and round number to the Rich console.

## Cost Tracking
- Call `litellm.completion_cost(response)` after every LLM call
- Accumulate into `OctagonState.total_session_cost`
- After each `moderator_node`: compare against `max_session_cost_usd`; terminate cleanly if exceeded

## Human Intervention
- Use LangGraph `interrupt` to pause the graph — not a custom async event or flag
- Print a specific, formatted missing-info request before pausing
- `state_dump.md` must be written on any unexpected exit, exception, or `KeyboardInterrupt`

## Console UI
- Use `rich.console.Console` + `rich.live.Live` — no print statements, no flicker
- Participant responses → Markdown blocks inside a named Panel
- Moderator events → distinct color/border: yellow (eviction), red (human request), green (consensus)
- Persistent dashboard: Current Round · Running Cost · Active Agents
