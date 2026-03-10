# ADR 002 — Use LiteLLM for All Model Interactions

**Date:** 2025-01-01  
**Status:** Accepted

---

## Context

Octagon must support participants running on different providers simultaneously: OpenAI, Anthropic, and local Ollama
models within the same debate session. Each provider has a different SDK, auth pattern, and response schema.

Options considered:

- **Direct SDK calls** (`openai`, `anthropic`, `ollama` packages) — requires conditional branching per provider in every
  node; adding a new provider means code changes throughout
- **LiteLLM** — single unified interface for 100+ models; handles auth, response normalisation, and cost tracking
  transparently; `completion_cost` built-in

## Decision

Use **LiteLLM** exclusively for all model calls, embeddings, and cost tracking. No direct provider SDK calls anywhere in
the codebase.

## Consequences

- `participant.py` and `moderator.py` pass `model_name` strings directly to LiteLLM — zero provider-specific logic in
  application code
- Adding a new provider requires only a `config.yaml` entry and a new env var — no code changes
- `litellm.embedding` is used for all cosine similarity scoring, keeping the embedding provider consistent with the chat
  provider per participant
- `litellm.completion_cost` gives per-call USD cost without maintaining a token-price lookup table
- LiteLLM's occasional breaking changes between versions must be managed; pin the version in `pyproject.toml`
