# Retrieval & Hybrid Search

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md) · drill the [retrieval question bank](../questions/retrieval.md)*

Retrieval is where the whole system's quality is capped. **A chunk that never makes top-k can't be grounded — most "hallucinations" are retrieval misses.** The interview tests whether you can pick a retriever deliberately, combine lexical + dense correctly, and know when *not* to add complexity.

---

## The three shapes of RAG — pick one before you build

"RAG" hides three different systems. Naming the shape sets your latency budget, footprint, and dominant lever:

| Shape | Example | Corpus / freshness | Dominant lever |
|---|---|---|---|
| **Chat-shaped** | support QA over a closed KB | bounded, comfortable latency | chunking, reranking, model choice |
| **Enterprise-search** | search across an org's private docs | large, permissioned, daily updates | isolation, ACLs, freshness |
| **Web-shaped** | Perplexity | billions of docs, seconds-fresh | the retrieval engine itself |

Conflating them is the classic way to over- or under-build. A 40k-doc support KB with a 1.5s budget is **chat-shaped** — spend your effort on indexing quality and reranking, not web-scale freshness machinery.

---

## Dense vs BM25 — different strengths

- **Dense (vector)** search captures *semantic* similarity — paraphrase, synonyms, concept match. It **blurs exact rare tokens**: error codes, part numbers, proper nouns.
- **BM25 (lexical)** matches exact tokens with a term-frequency / inverse-document-frequency score and length normalization. It's a **strong baseline** — the 2026 arXiv "From BM25 to Corrective RAG" benchmark found BM25 beats `text-embedding-3-large` on every metric *except* Recall@20.

> [!TIP]
> If a user searches for the exact error code "E1042", **BM25 inside a hybrid** surfaces it most reliably. A bigger embedding model won't help — exact rare tokens are dense retrieval's permanent weak spot.

---

## Hybrid search: fuse cheap recall, then rerank

Production systems (Perplexity, Uber, and most enterprise search) run BM25 + dense in parallel and **fuse** the results, because **recall and precision are different jobs**: cast a wide cheap net to maximize the chance the right doc is *present*, then spend an expensive [reranker](04-reranking-and-query-rewriting.md) to *order* the shortlist.

### RRF — Reciprocal Rank Fusion (the default)

RRF combines ranked lists using only *ranks*, not raw scores (which live on incomparable scales):

```
RRF_score(doc) = Σ  1 / (k + rank_i(doc))       # k ≈ 60, one term per retriever
```

RRF is robust and zero-maintenance — nothing to tune, nothing to drift.

### Weighted fusion — the senior caveat

You *can* beat RRF by a few nDCG points with a tuned `α·norm(bm25) + (1−α)·norm(dense)`. But **the weight is corpus/query-distribution-specific**: it goes stale on drift and needs a labelled set plus ongoing re-tuning. RRF's zero-maintenance robustness is usually worth more than the small gain. Mention this tradeoff — it's exactly the senior signal.

### Sparse-neural retrievers (the third option)

**SPLADE** and learned-sparse retrievers expand queries/docs into weighted term vectors — strong on *vocabulary mismatch with exact-match needs*. Useful, but unnecessary complexity for pure-paraphrase traffic over a clean corpus.

---

## The multi-stage pipeline companies actually run

```python
# Hybrid recall -> fuse -> hand off to the reranker.
def retrieve(query, k_recall=50, k_final=5):
    dense = vector_search(query, k=k_recall)     # semantic recall
    lexical = bm25_search(query, k=k_recall)     # exact-token recall
    fused = reciprocal_rank_fusion([dense, lexical], k=60)  # rank-based, no scale issues
    return rerank(query, fused, top_n=k_final)   # cross-encoder precision (04)
```

---

## When hybrid *isn't* worth it

Hybrid isn't free — an extra index, extra latency, and a BM25 analyzer to maintain. If the query distribution is **pure natural-language paraphrase over a small clean FAQ** with no codes or proper nouns, hybrid often buys nothing. **Measure first; pure dense may already be enough.** "Hybrid is always better" is a red-flag answer.

> [!WARNING]
> **The classic hybrid gotcha:** you turn on hybrid, but "ACME-X200" still misses even though the docs contain it. The cause is almost always the **BM25 analyzer splitting/stripping the hyphen** so the exact token never gets indexed. No fusion weight can surface a token BM25 never emitted. Check tokenization before you blame fusion or the encoder.

---

## Why not just stuff the whole corpus into a 1M-token context?

A favorite trap. Long context *loses* on four axes vs retrieval for corpus-scale QA:

1. **Cost** — you pay for every dumped token on *every* query.
2. **Latency** — prefill over a million tokens is slow.
3. **Recall** — "lost in the middle": mid-context facts get ignored.
4. **Attribution** — no chunk to cite.

Retrieve to keep the window small, cheap, ordered, and citable. Long context is a *complement* for holistic single-document tasks, not a replacement for retrieval.

---

## Interview angles

- **"Explain the parts of a RAG system."** → ingest/parse → chunk → embed → store → retrieve → rerank → generate → eval, with retrieval as the quality cap.
- **"How do you choose a retriever?"** → by query distribution: dense for paraphrase, BM25 for exact tokens, hybrid when you have both.
- **"What is hybrid search?"** → parallel BM25 + dense, fused (RRF), then reranked; recall and precision are different jobs.
- **"Why RRF over weighted fusion?"** → rank-based, scale-free, zero-maintenance; weighted `α` drifts and needs re-tuning.
- **"When is hybrid NOT worth it?"** → pure paraphrase over a clean FAQ — measure first.
- **"Why not a 1M-token context?"** → loses on cost, latency, recall (lost-in-the-middle), and attribution.

---

## 📚 Resources

- 📘 [Hybrid Search Explained](https://weaviate.io/blog/hybrid-search-explained) — Weaviate (Jan 2025) · the RRF walkthrough.
- 📄 [Hybrid Search in RAG: Dense + Sparse + BM25 + SPLADE + RRF](https://blog.gopenai.com/hybrid-search-in-rag-dense-sparse-bm25-splade-reciprocal-rank-fusion-and-when-to-use-which-fafe4fd6156e) — gopenai (Mar 2026) · when to use which.
- 📄 [From BM25 to Corrective RAG: Benchmarking Retrieval](https://arxiv.org/html/2604.01733v1) — arXiv (Apr 2026) · paper · BM25 beats `text-embedding-3-large` except Recall@20.
- 📘 [Hybrid search docs](https://docs.weaviate.io/weaviate/search/hybrid) — Weaviate · the API-level reference.
- 💻 [Danielskry/Awesome-RAG](https://github.com/Danielskry/Awesome-RAG) — Apache-2.0 · a curated index of retrieval papers/implementations. *Link + commentary.*
- 💻 [run-llama/llama_index](https://github.com/run-llama/llama_index) — **50.5k★, MIT** · retrievers, fusion, and query engines you can read for reference.
