# Reranking & Query Rewriting

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md) · drill the [reranking question bank](../questions/reranking.md)*

This is the **precision stage**. First-stage retrieval (dense + BM25) is tuned for recall — get the right chunk *somewhere* in the top 50, fast. But only 3–5 chunks fit the prompt, and packing it with marginal hits costs tokens and degrades the generator ("lost in the middle"). The reranker keeps the best few. Query rewriting improves the *input* to retrieval. Together they're the highest-ROI upgrades in RAG.

---

## Cross-encoder vs bi-encoder — the whole interview answer

The architectural distinction *is* the answer, so make it crisp:

| | Bi-encoder (first stage, recall) | Cross-encoder (rerank, precision) |
|---|---|---|
| How | `encode(query)`, `encode(doc)` **separately**, compare with cosine | `encode(query + doc)` **jointly**, full cross-attention → one score |
| Indexable? | yes — doc vectors computed offline | no — a fresh forward pass per query-doc pair |
| Speed | ms over millions (ANN) | ~ms **per pair** → shortlist only |
| Accuracy | approximate, scalable | accurate, ~1000× costlier |

A bi-encoder never lets the query and document "see" each other, so the match is approximate but pre-indexable (that's what makes search fast). A cross-encoder runs cross-attention over the concatenated pair — every query token attends to every document token — far more accurate, but you must run it fresh for *every* pair and can pre-compute nothing. That's the ~1000× cost and why it only runs on a shortlist.

> [!TIP]
> **"Why not just rerank the whole corpus / use a better embedding model?"** is *the* canonical reranking question. Answer: you cannot pre-index a cross-encoder, so scoring the whole corpus per query is seconds-to-minutes of compute; and a better bi-encoder still can't match pairwise attention. Two stages exist because recall is cheap-and-approximate and precision is costly-and-exact — spend the expensive model only on the shortlist.

---

## Two-stage retrieval under latency pressure

The pattern: bi-encoder fetches ~50 cheaply → cross-encoder re-scores those 50 → keep top 5. A reranker adds ~50–150ms — budget for it. Under tight latency, options are: rerank fewer candidates (top 30), use a faster/smaller reranker, or cache reranks for hot queries.

> [!WARNING]
> **A reranker cannot fix broken recall.** If `recall@50` is broken — the gold chunk often isn't retrieved at all — a reranker only *re-orders the shortlist*; it can't recover a chunk first-stage retrieval never fetched. Broken recall needs chunking / hybrid / ANN-param / ColBERT fixes first. Reranking fixes "in the set but ranked too low", not "not retrieved."

---

## ColBERT & late interaction — the middle ground

**ColBERT** stores a vector *per token* and scores with **MaxSim** (each query token matched to its best document token). It's more accurate than a bi-encoder and more scalable than a full cross-encoder, but the multi-vector index is much larger. Interview framing: "late-interaction — token-level matching, between bi-encoder speed and cross-encoder accuracy, at a storage cost."

---

## Query transformations

Improving the *query* often beats improving the retriever. Know the differences:

| Technique | What it does | When it helps |
|---|---|---|
| **Query rewriting** | clean/normalize/expand the raw query | conversational, messy, or under-specified queries |
| **Multi-query expansion** | generate N paraphrases, retrieve for each, fuse | recall-limited queries; hedges vocabulary mismatch |
| **Step-back prompting** | ask a more general question first, then specialize | reasoning-heavy queries needing broader context |
| **HyDE** | LLM writes a *hypothetical answer*, embed *that* to retrieve | zero-shot / domain-vocabulary mismatch |

### HyDE — and when it backfires

**HyDE** (Hypothetical Document Embeddings) generates a fake answer and retrieves against its embedding, closing the query-document vocabulary gap. It shines on zero-shot and domain-mismatch corpora. **But it can backfire on well-grounded factual lookups** — the hypothetical answer can hallucinate details that pull retrieval *away* from the true passage. On a clean grounded corpus with good recall, HyDE adds an LLM call and risk for little gain. "Add HyDE everywhere" is a red-flag answer.

---

## When GraphRAG / agentic / long-context are worth it (vs hype)

- **GraphRAG** — whole-corpus *sensemaking* ("main themes / who are the actors"), *not* narrow lookups. 5–10× pricier to index. → [06](06-advanced-rag.md)
- **Agentic multi-hop** — genuine multi-step questions, at 4–7× latency. Don't apply to single-hop lookups. → [06](06-advanced-rag.md)
- **Long context** — holistic single-document tasks, not corpus-scale QA. → [03](03-retrieval-and-hybrid-search.md)

For narrow factual lookups ("what's the refund window for plan X?"), **vector RAG + rerank beats all three** on quality and cost. Match the technique to the query distribution.

---

## Interview angles

- **"Why rerank on top of vector retrieval?"** → recall vs precision are different jobs; cross-encoder pairwise attention on the shortlist.
- **"Cross-encoder vs bi-encoder?"** → separate vs joint encoding; indexable-but-approximate vs accurate-but-per-pair.
- **"Two-stage retrieval under latency pressure?"** → fetch 50 cheap → rerank to 5; budget ~50–150ms; shrink candidates / smaller reranker / cache.
- **"Describe HyDE and when you'd use it."** → hypothetical answer → embed → retrieve; helps vocabulary mismatch, hurts grounded factual.
- **"Query rewriting vs multi-query vs step-back?"** → clean vs paraphrase-and-fuse vs generalize-then-specialize.

---

## 📚 Resources

- 📘 [Rerankers and Two-Stage Retrieval](https://www.pinecone.io/learn/series/rag/rerankers/) — Pinecone · the canonical reranking explainer.
- 🛠️ [Cohere Rerank](https://cohere.com/rerank) — Cohere · the managed reranker most teams reach for.
- 📄 [How Cohere Rerank-4 Improves RAG](https://orq.ai/blog/from-noise-to-signal-how-cohere-rerank-4-improves-rag) — orq.ai (Dec 2025) · measured lift.
- 📄 [Using LLMs as a Reranker for RAG](https://fin.ai/research/using-llms-as-a-reranker-for-rag-a-practical-guide/) — Fin (Sep 2025) · practical guide.
- 📄 [Flexible Training/Retrieval for Late Interaction (ColBERT/Jina)](https://arxiv.org/html/2508.03555v1) — arXiv (Aug 2025) · the late-interaction frontier.
- 📄 [HyDE: Precise Zero-Shot Dense Retrieval without Relevance Labels](https://arxiv.org/abs/2212.10496) — Gao et al. · the original HyDE paper.
- 📘 [HyDE (Haystack docs)](https://docs.haystack.deepset.ai/docs/hypothetical-document-embeddings-hyde) — deepset · implementation reference.
- 📄 [RAG II: Query Transformations](https://medium.com/thedeephub/rag-ii-query-transformations-49865bb0528c) — The Deep Hub · rewriting vs multi-query vs step-back.
