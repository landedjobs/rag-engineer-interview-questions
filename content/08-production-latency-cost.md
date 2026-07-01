# Production, Latency & Cost

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md) · drill the [production question bank](../questions/production.md)*

The production round tests whether you've *operated* a RAG system, not just built a demo. Interviewers probe latency budgets, cost levers, semantic caching, and — the highest-signal topic — **silent freshness drift**, the failure where every dashboard is green and answers are quietly wrong.

---

## The latency budget, stage by stage

Walk the pipeline and attribute a budget. A typical chat-shaped target of ~1.5s p95:

| Stage | Typical cost | Lever |
|---|---|---|
| Query rewrite / HyDE | +1 LLM call (~200–500ms) | skip when not needed |
| Embed query | ~10–30ms | batch, cache hot queries |
| Hybrid retrieve (ANN + BM25) | ~30–80ms | tune `efSearch`; pre-filter |
| Rerank (cross-encoder, ~50 cand) | ~50–150ms | fewer candidates / smaller model / cache |
| Generate | dominates; grows with output tokens | shorter output, streaming, model choice |

> [!TIP]
> **Output tokens cost ~4× input and dominate latency** (they're generated sequentially). A 2,000-in / 100-out call usually beats a 1,000-in / 1,000-out one on both cost and speed. Trim what you *ask the model to write*, not just what you feed it — and stream so time-to-first-token stays low even when total generation is long.

---

## Cost levers (the 2026 stack: 47–80% spend cut)

- **Semantic caching** (below) — serve repeat/similar queries without hitting retrieval + generation.
- **Tiered model routing** — cheap model for easy queries, strong/reasoning model only where needed.
- **Prompt caching** — a stable system prefix caches; contextual-retrieval indexing reuses the cached document (~90% cheaper).
- **Embedding-cost discipline** — batch ingestion, cache embeddings of *unchanged* docs, truncate dims (Matryoshka). When the embedding bill exceeds the LLM bill, this is usually why.
- **Pre-filtering** — metadata/ACL filters shrink the candidate set before expensive stages.

---

## Semantic caching

A semantic cache stores past `(query embedding → answer)` pairs and serves a cached answer when a new query is *semantically close enough* (cosine above a threshold) — not just an exact string match. It cuts both cost and latency for the long tail of near-duplicate questions.

```python
# Semantic cache: serve near-duplicate queries without re-running the pipeline.
def cached_answer(query, threshold=0.95):
    q = embed(query)
    hit = cache.nearest(q)                     # ANN over past query embeddings
    if hit and cosine(q, hit.embedding) >= threshold:
        return hit.answer                      # cache hit -> skip retrieve + generate
    ans = rag_pipeline(query)
    cache.put(q, ans)
    return ans
```

> [!WARNING]
> Set the threshold carefully: too loose and you serve a stale/wrong cached answer to a subtly different question; too tight and hit-rate collapses. Invalidate the cache on index updates, and never cache across tenants or permission boundaries.

---

## Silent freshness drift — the failure that defines the round

Latency, throughput, and individual retrievals all look healthy, dashboards are green — but users say answers are subtly out of date. This is **silent freshness drift**: stale documents, or a vendor embedding/model checkpoint change, decaying recall *distributionally* while every per-request metric stays fine.

You can't catch it with latency or error rate. You catch it with **freshness telemetry**:

- **Top-k-overlap on a fixed probe set** — run a frozen set of probe queries on a schedule; alert when the retrieved top-k drifts from the baseline.
- **Source-mtime vs chunk-embedded-mtime gap** — measure how stale the index is against the source of truth.
- **A `last_verified` timestamp** per doc + **CDC (change-data-capture) re-embedding** so edits re-index incrementally instead of a nightly full rebuild.

> [!WARNING]
> **Senior trap:** "answers got worse but we shipped nothing" is *not* random model variance. A sustained, one-directional quality drop with healthy infra metrics is the signature of a **stale index or a silent provider model update**. The strong design already has freshness telemetry + version pinning in place; "add more replicas" (a capacity fix) does nothing here.

---

## Scaling to 10M+ documents

The moves interviewers want to hear:

- **Index economics** — `N × dim × 4B × ~1.8` for HNSW; at 10M+ weigh **IVFPQ + reranker** to fit RAM (see [02](02-embeddings-and-vector-dbs.md)).
- **Sharding** — partition the index (by tenant, department, or hash) for both scale and isolation.
- **Caching** — semantic cache + prompt cache absorb the hot tail.
- **Pre-filtering** — narrow candidates with metadata before ANN.
- **CDC re-indexing** — keep a live 10M-doc index fresh incrementally, not with nightly full rebuilds.

---

## "It works in the demo — how is it production-ready?"

Production-ready is a **measurable bar**, not a vibe: it passes a split-eval gate on a **production-derived** golden set, clears injection tests, and meets the p99 latency/cost budget. "The answers are fluent and stakeholders are happy" is the demo; "it uses the latest models and a managed vector DB" is tooling, not correctness.

---

## Interview angles

- **"Optimize RAG latency in production."** → attribute a per-stage budget; trim output tokens; stream; pre-filter; cache; tiered routing.
- **"What is semantic caching?"** → cosine-threshold cache over query embeddings; cuts cost/latency on near-duplicates; invalidate on updates.
- **"Answers got worse but nothing shipped — why?"** → silent freshness drift / provider update; catch with top-k-overlap telemetry + pinned versions.
- **"Scale to 10M+ docs."** → IVFPQ + reranker, sharding, caching, pre-filtering, CDC re-indexing.
- **"How do you know it's production-ready?"** → split-eval gate on a production-derived golden set + injection tests + p99 budget.

---

## 📚 Resources

- 📄 [LLM Cost Optimization 2026](https://www.maviklabs.com/blog/llm-cost-optimization-2026) — Mavik Labs · routing + caching + batching → 47–80% cut.
- 💻 [zilliztech/GPTCache](https://github.com/zilliztech/gptcache) — Apache-2.0 · reference implementation of semantic caching. *Study + adapt with attribution.*
- 📄 [Rerankers and Two-Stage Retrieval](https://www.pinecone.io/learn/series/rag/rerankers/) — Pinecone · the latency/precision tradeoff of reranking.
- 📘 [Ragas: Context Precision](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) — the production eval metrics you gate on.
- 📄 [How to Crack the AI Engineer Interview 2026](https://aiengineeringinsider.substack.com/p/how-to-crack-the-ai-engineer-interview) — Substack · the production/eval round, described.
