# Octagon — Design Document

**Multi-Agent Technical Debate & Eviction Engine**  
Version 1.0 · Python 3.12+ · LangGraph + LiteLLM

---

## 1. Purpose

Octagon orchestrates a structured debate between multiple LLM participants managed by a Moderator. Given a technical
requirement — an architecture decision, API design, code structure — agents argue, critique, and refine each other's
proposals across multiple rounds until unanimous consensus is reached or session constraints are exhausted.

The final output is `octagon_result.md`, a single file containing the agreed-upon design or the best available state
when limits were hit.

---

## 2. Project Structure

```
octagon/
│
├── docs/                          # Human-facing documentation
│   ├── DESIGN.md                  # ← this file
│   ├── RUNBOOK.md                 # Auth, env vars, build, troubleshooting
│   └── decisions/                 # Architecture Decision Records
│       ├── 001-langgraph-stategraph.md
│       ├── 002-litellm-abstraction.md
│       └── 003-eviction-scoring.md
│
├── octagon/                       # Application source
│   ├── __init__.py
│   ├── main.py                    # CLI entry point
│   │
│   ├── graph/
│   │   ├── builder.py             # StateGraph construction
│   │   ├── nodes.py               # All node functions
│   │   └── edges.py               # Routing / conditional edges
│   │
│   ├── agents/
│   │   ├── moderator.py           # Moderator LLM logic
│   │   └── participant.py         # Participant LLM logic
│   │
│   ├── core/
│   │   ├── state.py               # Pydantic v2 State schema
│   │   ├── config.py              # config.yaml loader + auth validation
│   │   ├── eviction.py            # Eviction scoring engine
│   │   └── cost.py                # LiteLLM cost tracking
│   │
│   ├── ui/
│   │   └── dashboard.py           # Rich console / Live display
│   │
│   └── utils/
│       └── similarity.py          # Cosine similarity via litellm.embedding
│
├── tests/
│   ├── test_eviction.py
│   ├── test_state.py
│   └── test_graph.py
│
├── .env                           # Actual secrets (gitignored)
├── .env.example                   # Env var template (committed)
├── AGENTS.md                      # Coding rules and logic specs
├── CLAUDE.md                      # Project context
├── README.md
├── config.yaml                    # Runtime config (operators edit this)
└── pyproject.toml
```

---

## 3. Data Schemas (Pydantic v2)

### `OctagonState`

The single shared object threaded through every node in the `StateGraph`.

| Field                  | Type                   | Description                           |
|------------------------|------------------------|---------------------------------------|
| `messages`             | `list[Message]`        | Full debate history in order          |
| `active_participants`  | `list[Participant]`    | Currently live agents                 |
| `evicted_participants` | `list[EvictionRecord]` | Who was removed, why, and when        |
| `total_session_cost`   | `float`                | Cumulative USD spend                  |
| `current_round`        | `int`                  | 1-indexed round counter               |
| `awaiting_human`       | `bool`                 | True when graph is paused for input   |
| `human_attempts`       | `int`                  | Number of human input attempts so far |

### `ParticipantConfig` (from `config.yaml`)

| Field            | Type  | Description                                           |
|------------------|-------|-------------------------------------------------------|
| `name`           | `str` | Display name shown in console                         |
| `model_name`     | `str` | LiteLLM model string (e.g. `gpt-4o`, `ollama/llama3`) |
| `provider`       | `str` | `openai` / `anthropic` / `ollama`                     |
| `persona_prompt` | `str` | System prompt defining this agent's role              |
| `token_limit`    | `int` | Cumulative token ceiling before eviction              |

### `EvictionRecord`

| Field              | Type    | Description                                  |
|--------------------|---------|----------------------------------------------|
| `participant_name` | `str`   | Who was evicted                              |
| `reason`           | `str`   | `deviation` / `repetition` / `token_limit`   |
| `score`            | `float` | The exact value that triggered the threshold |
| `round`            | `int`   | Round number when eviction occurred          |

---

## 4. Graph Architecture

```
                    ┌─────────────┐
          ┌────────►│  broadcast  │
          │         │    _node    │
          │         └──────┬──────┘
          │                │
          │         ┌──────▼──────┐
          │         │ participant │  ← runs once per active agent, async
          │         │    _node    │
          │         └──────┬──────┘
          │                │
          │         ┌──────▼──────┐      ┌─────────────┐
          │         │  moderator  │─────►│  evict_node │
          │         │    _node    │      └─────────────┘
          │         └──────┬──────┘
          │                │
          │    ┌───────────┼────────────┐
          │    │           │            │
          │  needs      consensus    max rounds /
          │  info       reached      budget hit
          │    │           │            │
          │  ┌─▼──────┐  ┌─▼───────┐  ┌─▼───────┐
          │  │ human  │  │  write  │  │  write  │
          │  │ input  │  │ result  │  │ result  │
          │  │ _node  │  └─────────┘  └─────────┘
          │  └───┬────┘
          │      │ answered
          └──────┘
               │ timeout / max attempts
           ┌───▼──────┐
           │state_dump│
           └──────────┘
```

### Node Responsibilities

| Node               | File             | Responsibility                                                                                                       |
|--------------------|------------------|----------------------------------------------------------------------------------------------------------------------|
| `broadcast_node`   | `graph/nodes.py` | Builds a context packet per participant: moderator summary + all peer viewpoints from the previous round             |
| `participant_node` | `graph/nodes.py` | Async LiteLLM call per active agent; appends response to `messages`; tracks cumulative token count                   |
| `moderator_node`   | `graph/nodes.py` | Evaluates all responses via eviction scoring; checks consensus; checks cost budget; determines next edge             |
| `evict_node`       | `graph/nodes.py` | Removes participant from `active_participants`; appends `EvictionRecord`; logs name + reason + score to Rich console |
| `human_input_node` | `graph/nodes.py` | Pauses graph via LangGraph `interrupt`; prints missing-info request; reads stdin; resumes or triggers state dump     |

### Edge Routing (`graph/edges.py`)

After `moderator_node`, a conditional edge routes to one of:

- `broadcast_node` — continue debate
- `evict_node` — one or more participants need removal first, then back to `broadcast_node`
- `human_input_node` — moderator flagged insufficient information
- `END` — consensus reached, max rounds hit, or budget exceeded

---

## 5. Eviction Scoring (`core/eviction.py`)

All similarity scoring uses `litellm.embedding` with cosine similarity.

| Trigger             | Comparison                                          | Threshold                                 | Action |
|---------------------|-----------------------------------------------------|-------------------------------------------|--------|
| **Topic deviation** | Response vs. original requirement embedding         | similarity < `deviation_threshold` (0.4)  | Evict  |
| **Repetition**      | Response vs. participant's last 2 responses         | similarity > `repetition_threshold` (0.9) | Evict  |
| **Token overrun**   | Cumulative tokens vs. `token_limit` per participant | exceeded                                  | Evict  |

**Moderator transparency rule:** On every eviction the console must display:

- Participant name
- Violation type
- Exact score vs. threshold
- Round number

---

## 6. Cost Tracking (`core/cost.py`)

- Call `litellm.completion_cost(completion_response)` after every LLM call
- Accumulate into `OctagonState.total_session_cost`
- After each `moderator_node` execution, compare against `max_session_cost_usd`
- If exceeded: route to `END`, write `octagon_result.md` with current best state

---

## 7. Human Intervention (`graph/nodes.py` → `human_input_node`)

Triggered when the Moderator detects it cannot proceed without more information.

1. Graph pauses via LangGraph `interrupt`
2. Console prints a specific, formatted request for the missing information
3. Wait for stdin input
4. If answered: resume from `broadcast_node` with enriched state
5. If no answer or blank: increment `human_attempts`
6. If `human_attempts` >= `max_human_attempts`: write `state_dump.md` and exit

`state_dump.md` is also written on any uncaught exception or `KeyboardInterrupt`.

---

## 8. Console UI (`ui/dashboard.py`)

Built with `rich.console.Console` and `rich.live.Live` to prevent flicker.

| Element                   | Rendering                                                        |
|---------------------------|------------------------------------------------------------------|
| **Session dashboard**     | Persistent top bar: `Round N/10 · $0.43 spent · 3 agents active` |
| **Participant responses** | Markdown blocks inside a Panel, agent name as title              |
| **Eviction events**       | Yellow bordered panel: name, reason, score                       |
| **Human input requests**  | Red bordered panel with specific missing-info text               |
| **Consensus reached**     | Green bordered panel with summary                                |

---

## 9. Termination & Output Files

| Condition                       | Output file         | Contents                        |
|---------------------------------|---------------------|---------------------------------|
| Unanimous agreement             | `octagon_result.md` | Full consensus design           |
| `max_rounds` reached            | `octagon_result.md` | Best available state + note     |
| `max_session_cost_usd` exceeded | `octagon_result.md` | Partial result + cost summary   |
| Human input timeout             | `state_dump.md`     | Full state + message history    |
| Unexpected exception            | `state_dump.md`     | Full state + traceback          |
| Manual `Ctrl+C`                 | `state_dump.md`     | Full state at time of interrupt |

---

## 10. Async Model

All LLM I/O is `async`. Participant nodes run concurrently per round using `asyncio.gather`. The graph itself is invoked
with `asyncio.run(graph.ainvoke(...))`.

```python
# Concurrent participant calls within one round
responses = await asyncio.gather(*[
    call_participant(p, context) for p in state.active_participants
])
```
