# The RAG Interview Question Bank

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md)*

**100+ questions**, topic-grouped and difficulty-tiered. Two kinds:

- **Multiple-choice with per-option explanations** — sourced from the Landed RAG course checkpoints. The wrong options teach as much as the right one; each has a *why*. Read the ▫️ distractors — they're the traps interviewers set.
- **Open / system-design questions** — the 33 real-and-representative questions gathered from 2026 interview loops, each with a **provenance label**:
  - **✅ Reported** — attested in a public source (linked).
  - **🔮 Representative** — a realistic question in the style of the loop, not attributed to a specific report.

## Difficulty tiers

| Tier | Meaning |
|---|---|
| 🟢 **Core** | you must get these right — foundations every RAG role assumes |
| 🟡 **Senior** | tradeoff fluency, production reasoning, "what's the cheapest lever" |
| 🔴 **Staff** | index economics, isolation, adversarial security, judge calibration |

## By topic

| Topic | File | Focus |
|---|---|---|
| Retrieval & architecture | [retrieval.md](retrieval.md) | shapes of RAG, dense vs BM25, hybrid, RRF, long-context |
| Embeddings & vector stores | [embeddings.md](embeddings.md) | model choice, ANN/HNSW, index sizing, shearing, DB choice |
| Chunking | [chunking.md](chunking.md) | size by metric, parsing, orphaned chunks, contextual/late, small-to-big |
| Reranking & query rewriting | [reranking.md](reranking.md) | cross vs bi-encoder, two-stage, ColBERT, HyDE, GraphRAG/agentic |
| Evaluation | [evaluation.md](evaluation.md) | IR metrics, split eval, faithfulness limits, golden sets, LLM-judge κ |
| Production, latency & cost | [production.md](production.md) | latency budget, semantic caching, freshness drift, scaling, multi-turn |
| Security & enterprise | [security.md](security.md) | retrieval-time ACLs, PII, multi-tenancy, injection, audit, posture |

## How to drill

1. Cover the options, answer aloud, *then* read the explanations.
2. For every ▫️ wrong option, articulate *why* it's wrong before reading the *why*.
3. For open questions, practice the **structure** first: clarify requirements → name the metric/stage → propose the cheapest lever → state the tradeoff.
4. Pair each topic file with its [content mini-lecture](../content/) for the mechanism.

> [!TIP]
> The meta-skill every tier rewards: **localize before you fix, name a metric, and state the tradeoff.** "Use a bigger model" with no metric attached is the answer that ends the round.

---

*Count: **53** multiple-choice checkpoint questions (with per-option explanations) + **43** reported/representative open questions + **4** worked system-design prompts = **100** and growing. Contributions welcome — see [CONTRIBUTING.md](../CONTRIBUTING.md).*
