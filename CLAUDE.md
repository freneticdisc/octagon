# Octagon — Project Context

## Stack
- Python 3.12+, `uv` (use `uv add`), `uvx`-compatible
- Core: `langgraph`, `litellm`, `pydantic` (v2), `pyyaml`, `rich`

## State Schema
```python
messages: list            # full debate history
active_participants: list # current persona configs
evicted_participants: list # {name, reason, score, round}
total_session_cost: float
current_round: int
awaiting_human: bool
human_attempts: int
```

## Config (`config.yaml`)
Per-participant: `model_name`, `provider` (openai/anthropic/ollama), `persona_prompt`, `token_limit`  
Global: `max_rounds` (default 10), `max_session_cost_usd`, `max_human_attempts`, `deviation_threshold` (0.4), `repetition_threshold` (0.9)

## Graph Nodes (LangGraph `StateGraph`)
1. `broadcast_node` — send moderator summary + all peer views to each participant
2. `participant_node` — async LiteLLM call per active participant
3. `moderator_node` — run eviction scoring + consensus check + cost check
4. `evict_node` — remove participant, log name + reason + score to console
5. `human_input_node` — pause graph via `interrupt`, request missing info via stdin; on timeout write `state_dump.md` and exit

## Key Files
- `octagon/graph/nodes.py` — all node implementations
- `octagon/graph/edges.py` — conditional routing logic
- `octagon/core/eviction.py` — scoring engine
- `octagon/core/config.py` — YAML loader + auth validation
- `octagon/ui/dashboard.py` — Rich console display

## Termination
- Unanimous agreement → `octagon_result.md`
- `max_rounds` or `max_session_cost_usd` exceeded → `octagon_result.md`
- Insufficient info after `max_human_attempts` → `state_dump.md`
- Any uncaught exception or `KeyboardInterrupt` → `state_dump.md`

## Commands
```
uv run octagon   # run engine
uv run pytest    # tests
uv lock          # lock deps
uv add <pkg>     # add dependency
```
