# ADR 003 — Use Embedding Cosine Similarity for Eviction Scoring

**Date:** 2025-01-01  
**Status:** Accepted

---

## Context

Octagon needs to detect two failure modes in participant responses:

1. **Topic deviation** — the participant is arguing about something unrelated to the original requirement
2. **Repetition** — the participant is recycling the same argument without adding value

Options considered for scoring:

- **LLM-as-judge** — ask a secondary LLM to score each response; highly accurate but adds cost and latency per participant per round; creates a recursive dependency
- **Keyword matching / TF-IDF** — fast and cheap, but brittle; misses semantic drift where wording changes but meaning repeats
- **Embedding cosine similarity** — encodes semantic meaning into a vector; cosine distance captures both deviation and repetition accurately; cost is low (embedding calls are ~10x cheaper than completion calls); already available via LiteLLM

## Decision

Use **`litellm.embedding` + cosine similarity** for all eviction scoring.

- Topic deviation: compare current response embedding vs. original requirement embedding
- Repetition: compare current response embedding vs. participant's last 2 response embeddings

## Thresholds

| Check | Default | Direction |
|---|---|---|
| Deviation | 0.4 | evict if similarity **below** threshold |
| Repetition | 0.9 | evict if similarity **above** threshold |

Thresholds are configurable in `config.yaml` — operators can tune for debate aggressiveness.

## Consequences

- Embedding calls add latency per participant per round; acceptable given they are non-blocking async calls
- The same embedding model is used for all participants for consistency; configurable at the global level
- Thresholds are heuristics and may need tuning per use case — the ADR records the defaults but does not prescribe them as final
