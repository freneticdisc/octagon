# ADR 001 — Use LangGraph StateGraph for Orchestration

**Date:** 2025-01-01  
**Status:** Accepted

---

## Context

Octagon requires a cyclic debate loop with conditional branching (continue, evict, pause for human, terminate). The loop must be interruptible mid-execution for human input and must persist state across node transitions.

Options considered:

- **Plain `asyncio` loop** — simple, but no built-in state management, no interrupt mechanism, difficult to add new branches without restructuring
- **Celery** — designed for distributed task queues, not stateful sequential graphs; significant operational overhead
- **LangGraph `StateGraph`** — purpose-built for cyclic, stateful LLM workflows; native `interrupt` for human-in-the-loop; explicit edge routing

## Decision

Use **LangGraph `StateGraph`** as the sole orchestration mechanism.

## Consequences

- State is a single typed Pydantic object passed through every node — no global variables, no implicit shared state
- Conditional edges in `graph/edges.py` make all branching logic explicit and testable
- LangGraph's `interrupt` primitive handles human-in-the-loop without custom async event machinery
- Adds `langgraph` as a required dependency; acceptable given it is the primary value-add of the framework
