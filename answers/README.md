# Worked RAG System Designs

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md)*

One reusable [**rubric**](rag-system-design-rubric.md), applied across four original worked designs. Each follows the same structure — clarify → three swim-lanes → defend each stage with a metric and a tradeoff → name the silent failures — so you can internalize the *shape* of a strong answer, not memorize one script.

| Design | The core challenge it drills |
|---|---|
| [QA over 10M documents](worked-qa-over-10m-docs.md) | index economics, sharding, caching, freshness at scale |
| [Enterprise multi-tenant RAG](worked-enterprise-multi-tenant-rag.md) | per-tenant isolation, retrieval-time ACLs, the recall cliff |
| [Agentic RAG over tools](worked-agentic-rag-over-tools.md) | when to loop, stop conditions, trajectory eval, tool safety |
| [Private RAG for a regulated (HIPAA) customer](worked-private-rag-regulated.md) | control plane vs data plane, PII, deployment posture, audit |

## How to use these

1. Read the [rubric](rag-system-design-rubric.md) first — the 11-stage scorecard and the "clarify before you draw" opener.
2. For each design, **cover the answer and try it yourself** against the rubric before reading.
3. Time yourself: a real round is 45–60 minutes. Practice narrating the three swim-lanes (ingest / query / gate) out loud.
4. The differentiator is always the same: name a metric per stage, state the tradeoff, and surface the silent failures *before* the interviewer asks.
