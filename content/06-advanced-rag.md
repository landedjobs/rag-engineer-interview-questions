# Advanced RAG — GraphRAG, Agentic RAG, Contextual Retrieval, Multimodal

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md)*

By 2026 the advanced-RAG techniques are interview table stakes — but the *senior* answer is almost always "here's when it's worth it, and here's when vanilla vector RAG + rerank still wins." Interviewers use these to separate people who match a technique to a query distribution from people who reach for the newest acronym.

---

## Contextual Retrieval (Anthropic, Sep 2024)

The most bang-for-buck advanced technique. Prepend an LLM-written context to each chunk *before* indexing it (into embeddings **and** BM25), so isolated chunks keep the identifiers they'd otherwise lose (see [01-chunking](01-chunking.md)). Prompt-cache the whole document once → ~90% cheaper. Measured: **~49% fewer failed retrievals, ~67% with a reranker.** If you learn one advanced technique, learn this — it's the answer to the orphaned-chunk question.

---

## GraphRAG (Microsoft, 34k★ MIT)

GraphRAG builds a **knowledge graph** from the corpus (entities + relationships + community summaries) and retrieves over graph structure instead of flat chunks.

- **Strength:** whole-corpus **sensemaking** — "what are the main themes?", "who are the key actors and how are they connected?" — questions no single chunk answers.
- **Cost:** the graph-construction indexing pass is **5–10× pricier** than embedding chunks.
- **Weakness:** narrow factual lookups. On "what's the refund window for plan X?", vector RAG + rerank beats GraphRAG on both quality and cost.

> [!WARNING]
> **"GraphRAG is strictly better than vector RAG" is a red-flag answer.** Match the technique to the query distribution: reserve GraphRAG for sensemaking across a corpus; keep vector RAG + rerank for lookups.

---

## Agentic RAG — the 2026 default for multi-step queries

Instead of a fixed retrieve → generate pipeline, an **agent** loops: retrieve → reason → decide (answer, re-retrieve with a new query, or call a tool) → repeat. It's the default for genuinely multi-step questions in 2026 (LangGraph, LlamaIndex agents).

Two foundational patterns to name:

- **Self-RAG** ([arXiv 2310.11511](https://arxiv.org/abs/2310.11511)) — the model emits reflection tokens to decide *when* to retrieve and whether retrieved passages are relevant/supported.
- **CRAG (Corrective RAG)** ([arXiv 2401.15884](https://arxiv.org/abs/2401.15884)) — a lightweight evaluator grades retrieval quality; on low confidence it triggers a web search or query rewrite instead of answering from bad context.

> [!WARNING]
> **Agentic RAG costs 4–7× the latency of a single retrieval and can get loop-stuck.** For single-hop lookups it's pure overhead. The senior answer names the tradeoff and a stop condition (max steps, budget cap, confidence threshold), and reserves agentic loops for questions that actually need multi-hop reasoning.

```python
# Corrective/agentic loop with a hard stop -- don't let it spin.
def agentic_rag(query, max_steps=3):
    for step in range(max_steps):
        ctx = retrieve(query)
        grade = evaluate_relevance(query, ctx)      # CRAG-style confidence
        if grade == "sufficient":
            return generate(query, ctx)
        query = rewrite_or_web_search(query, ctx)    # correct, then retry
    return generate(query, ctx, note="low-confidence")  # stop condition, not infinite loop
```

---

## Multimodal RAG

Retrieval over images, tables, charts, and PDFs — not just text. Two approaches:

1. **Unified multimodal embeddings** (e.g. Qwen3-VL, CLIP-style) — embed text and images into one space, retrieve across both.
2. **Parse-then-embed** — extract structure (tables → Markdown, images → captions/summaries) and embed the text representation. More robust for tables/charts where layout carries meaning.

Interview framing: for tables and charts, structure-preserving parsing usually beats a raw image embedding — the numbers and headers matter, and a caption or Markdown table retrieves more reliably than a pixel patch.

---

## When to stay vanilla

For a corpus of **narrow factual lookups over clean docs**, the strongest system is often the boring one: structure-aware + contextual chunks, hybrid BM25 + dense with RRF, a cross-encoder reranker, grounded cited generation, and a real eval gate. GraphRAG, agentic loops, and long-context are *complements* for specific query shapes — not upgrades you apply everywhere. Saying this is a senior signal.

---

## Interview angles

- **"When is GraphRAG worth it?"** → whole-corpus sensemaking, not narrow lookups; 5–10× indexing cost.
- **"When is agentic RAG the *wrong* tool?"** → single-hop lookups — 4–7× latency for no benefit; need a stop condition.
- **"Explain Self-RAG / CRAG."** → reflection-token retrieval decisions / confidence-graded corrective retrieval.
- **"How would you do multimodal RAG over financial PDFs?"** → structure-preserving parse (tables → Markdown) then embed; layout carries meaning.

---

## 📚 Resources

- 🧑‍🏫 [Contextual Retrieval in AI Systems](https://www.anthropic.com/engineering/contextual-retrieval) — Anthropic (Sep 2024) · the ~49%/~67% technique.
- 💻 [microsoft/graphrag](https://github.com/microsoft/graphrag) — **34k★, MIT** · the canonical graph-shaped retrieval implementation. *Adaptable with attribution.*
- 📘 [Welcome to GraphRAG](https://microsoft.github.io/graphrag/) — Microsoft · the docs + methodology.
- 📄 [Agentic RAG Patterns 2026](https://www.digitalapplied.com/blog/agentic-rag-patterns-multi-step-reasoning-guide) — digitalapplied (Apr 2026) · multi-step reasoning patterns.
- 📄 [Next-Gen Agentic RAG with LangGraph (2026)](https://medium.com/@vinodkrane/next-generation-agentic-rag-with-langgraph-2026-edition-d1c4c068d2b8) — the LangGraph agentic build.
- 📄 [Self-RAG](https://arxiv.org/abs/2310.11511) · [CRAG](https://arxiv.org/abs/2401.15884) — arXiv · the two foundational agentic-RAG papers.
- 📄 [From RAG to GraphRAG](https://www.alexanderthamm.com/en/blog/from-rag-to-graphrag/) — Alexander Thamm (Jul 2025) · when graph retrieval earns its cost.
- 💻 [NirDiamant/RAG_Techniques](https://github.com/NirDiamant/RAG_Techniques) — **28k★, Apache-2.0** · notebooks for GraphRAG, agentic RAG, contextual retrieval, HyDE. *Study + adapt with attribution.*
