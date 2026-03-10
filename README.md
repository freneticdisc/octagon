# Octagon

> A multi-agent technical debate and eviction engine. LLMs argue, critique, and refine until consensus is reached.

Given a technical requirement, Octagon spawns configurable LLM participants under a Moderator. Agents debate in rounds.
Underperformers get evicted. The winner is a consensus design written to `octagon_result.md`.

---

## Quickstart

```bash
# Copy and fill in your API keys
cp .env.example .env

# Install dependencies
uv sync

# Run a debate
uv run octagon "Design a rate-limiting architecture for a public API"

# Or pipe a markdown spec
uv run octagon < requirements.md

# No install required (uvx)
uvx octagon "..."
```

## Documentation

| Doc                                  | Purpose                                        |
|--------------------------------------|------------------------------------------------|
| [`docs/DESIGN.md`](docs/DESIGN.md)   | Architecture, graph, schemas, eviction logic   |
| [`docs/RUNBOOK.md`](docs/RUNBOOK.md) | Auth, env vars, build, deploy, troubleshooting |
| [`docs/decisions/`](docs/decisions/) | Architecture Decision Records (ADRs)           |

## Configuration

Edit [`config.yaml`](config.yaml) to configure participants, models, personas, and session limits. See [
`docs/RUNBOOK.md`](docs/RUNBOOK.md) for provider authentication.

## Output Files

| File                | Written when                                              |
|---------------------|-----------------------------------------------------------|
| `octagon_result.md` | Consensus reached or max rounds hit                       |
| `state_dump.md`     | Unexpected exit, manual interrupt, or human input timeout |

---

**Stack:** Python 3.12+ · LangGraph · LiteLLM · Pydantic v2 · Rich
