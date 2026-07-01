# Embeddings & Vector Databases

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md) · drill the [embeddings question bank](../questions/embeddings.md)*

This is the stage where candidates reach for "a bigger embedding model" and seniors reach for a metric. The interview tests whether you understand embeddings + ANN *mechanically*, can size the index on a whiteboard (RAM, cost, latency), and choose a vector store as a *procurement decision* — not a brand loyalty.

---

## Choosing an embedding model: domain-fit first, size last

Model size is rarely the dominant lever; **domain fit, chunking, and hybrid search usually move retrieval far more.** The selection workflow:

1. **Start from MTEB** ([leaderboard](https://huggingface.co/spaces/mteb/leaderboard)) — the canonical embedding benchmark. Mid-2026, Gemini Embedding 2 leads the general leaderboard; Qwen3-VL-2B is a top cross-modal OSS option.
2. **Filter by your constraints:** dimension (a RAM bill — see below), context length, multilingual/multimodal need, and whether you can self-host.
3. **Validate on *your* corpus** with a labelled set — MTEB rank does not guarantee rank on your domain. A domain-tuned smaller model often beats a bigger general one.

> [!TIP]
> **Matryoshka embeddings** let you truncate dimensions (e.g. 1536 → 512) at a *measured* recall cost you can recover with a reranker — no re-embed. This is the clean lever when the index won't fit in RAM.

### Where embeddings fail (a favorite interview probe)

Dense embeddings blur exact rare tokens and struggle with **negation, temporal reasoning, and precise identifiers** ("E1042", "ACME-X200"). This is *why* you keep BM25 in a hybrid ([03](03-retrieval-and-hybrid-search.md)): lexical search catches exactly what embeddings blur.

---

## Cosine, dot, L2 — and why cosine

Cosine ignores magnitude, so document length doesn't distort meaning. On **normalized** vectors, cosine, dot product, and L2 rank identically — pick whatever the index supports. The interview answer: "cosine because it's magnitude-invariant; on normalized vectors it's equivalent to dot/L2 for ranking."

---

## ANN, HNSW, and the recall ceiling you can tune for free

At 100M vectors, exact (brute-force) search is `O(N·d)` — seconds slow. **Approximate Nearest Neighbor (ANN)** buys 100–1000× speed for a few points of recall. The dominant index is **HNSW** (Hierarchical Navigable Small World): a layered proximity graph you descend greedily. Two knobs matter:

- **`M`** — edges per node. More edges = better recall, more RAM.
- **`efSearch`** — candidates explored at query time. Higher = better recall, more latency.

> [!WARNING]
> **Senior trap:** if `recall@20` is stuck and chunking hasn't moved it, *raise `efSearch` (and/or `M`) before swapping the embedding model.* HNSW is approximate — a low `efSearch` silently drops true top-k neighbors. The index config may be your recall ceiling, and it's free to change. A model swap is a full re-embed and rarely the cheapest lever.

---

## Sizing the index on a whiteboard

Interviewers love this because you can do it from dimension and count alone:

```
RAM ≈ N × dim × 4 bytes × (~1.5–2.0 for the HNSW graph)

Example: 50M chunks × 1,536-d × 4 bytes ≈ 300 GB raw
         × ~1.8 for HNSW ≈ 450–600 GB
```

If that doesn't fit your budget, the moves that **don't require re-embedding**:

- **Matryoshka dimension truncation** — shrink the stored vector in place.
- **Product quantization (IVFPQ)** — ~20× less memory (~70 GB vs ~1.4 TB at 1B vectors), at a recall cost you recover with a reranker on the shortlist.

At **800M+ vectors on a tight RAM budget**, "HNSW, best recall, accept the bill" (~1.4 TB) may simply not fit — **IVFPQ + reranker** is the procurement-correct answer.

---

## Representation shearing — the silent embedding-swap failure

Vectors from two different models live in **different spaces** and are *not* cosine-comparable. If you upgrade your embedding model and push new vectors into the existing live index alongside the old ones, **recall shears silently** — cross-generation comparisons are meaningless. The safe path: full re-embed into a **parallel (dual) index**, validate on a held-out set, then **atomic cutover** with instant rollback.

```python
# WRONG: in-place, mixes generations -> silent recall shear.
# RIGHT: dual index, validate, atomic cutover.
new_index = build_index(embed=embed_v2)        # parallel, offline
assert recall_at_k(eval_set, retrieve_new, 20) >= baseline
cutover(traffic_to=new_index)                  # atomic; keep old for rollback
```

---

## The vector-database decision (no single "best")

Choosing a store is a **procurement decision** on cost, ops, memory/recall, isolation, and residency — not a brand. Approximate mid-2026 numbers:

| Store | Latency (p50) / QPS | When to pick it | License / model |
|---|---|---|---|
| **Qdrant** | ~30–40ms / 8–15k | fastest OSS/managed; strong filtering | Apache-2.0, self-host or managed |
| **Pinecone** | balanced | best all-round managed; ship fast | proprietary, managed |
| **pgvector** | depends | you already run Postgres; multi-tenant **RLS** | PostgreSQL license |
| **FAISS** | ~10–20ms / 20–50k | unmanaged max throughput; you own ops | MIT, library |
| **Milvus** | ~50–80ms / 10–20k | scale + hybrid, heavier ops | Apache-2.0 |
| **Weaviate** | ~50–70ms / 3–8k | built-in hybrid + modules | BSD-3, self-host or managed |

> [!TIP]
> **Do you even need a vector DB?** For a few thousand chunks, a NumPy matrix + brute-force cosine (or FAISS flat) is fine and simpler. Vector DBs earn their keep at scale, with filtering, and with multi-tenant isolation. Say this — it signals you don't cargo-cult infrastructure.

**Build vs buy:** below ~10M vectors, **managed is cheaper and faster to ship**. Self-host (Vespa/Milvus/FAISS) when you have the ops muscle or a hard constraint. A 4-person startup with a data-residency requirement should **buy a managed store with per-tenant isolation / in-region (VPC) deployment** — residency is satisfied by isolation/region features, not by self-hosting everything.

---

## Interview angles

- **"What vector DBs have you used and why?"** → name the *tradeoff axis*, not a brand; tie the pick to scale/isolation/residency.
- **"Do you need a vector database for RAG?"** → no, below a few thousand chunks; they earn their keep at scale + filtering + isolation.
- **"Where do embeddings fail?"** → exact rare tokens, negation, temporal — hence hybrid with BM25.
- **"How does ANN / HNSW work?"** → greedy descent through a layered proximity graph; `M` and `efSearch` trade recall for RAM/latency.
- **"Estimate index RAM for 50M chunks."** → `N × dim × 4B ≈ 300 GB` raw at 1,536-d, ×~1.8 for HNSW; then Matryoshka/PQ to cut it.
- **"You swapped encoders and recall dropped — why?"** → representation shearing; full re-embed behind a dual index.

---

## 📚 Resources

- 🛠️ [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) — HuggingFace · the canonical embedding selector.
- 📄 [Best Embedding Model for RAG 2026: 10 Models](https://milvus.io/blog/choose-embedding-model-rag-2026.md) — Milvus (Mar 2026) · Gemini Embedding 2 best all-rounder, Qwen3-VL-2B top cross-modal OSS.
- 📄 [Best Vector Databases 2026](https://www.firecrawl.dev/blog/best-vector-databases) — Firecrawl (May 2026) · the current landscape.
- 📄 [Pinecone vs Weaviate vs Qdrant vs pgvector](https://openhelm.ai/blog/pinecone-vs-weaviate-vs-qdrant-vs-pgvector) — openhelm.ai (Sep 2025) · the decision, quantified.
- 📄 [Pinecone vs Weaviate vs Qdrant vs Milvus](https://tensorblue.com/blog/vector-database-comparison-pinecone-weaviate-qdrant-milvus-2025) — tensorblue · the latency/QPS table used above.
- 📄 [10 Best Vector DBs for RAG](https://www.zenml.io/blog/vector-databases-for-rag) — ZenML (Oct 2025) · procurement-angle roundup.
- 📄 [Long-Context Retrieval with Monarch Mixer](https://hazyresearch.stanford.edu/blog/2024-01-11-m2-bert-retrieval) — Stanford Hazy · the long-context embedding frontier.
