# Reranking & Query Rewriting — Questions

> *Part of the [RAG question bank](README.md) · pair with [content/04-reranking-and-query-rewriting.md](../content/04-reranking-and-query-rewriting.md)*

Legend: ✅ correct · ▫️ distractor · 🟢 Core · 🟡 Senior · 🔴 Staff · ✅ Reported / 🔮 Representative

---

## Multiple-choice checkpoints

**Q1 · 🟡 Why not just run the cross-encoder reranker over the whole corpus and skip first-stage retrieval?**

- ▫️ Cross-encoders are less accurate than bi-encoders — *They're more accurate — that's why we rerank with them.*
- ✅ It's far too slow — a cross-encoder must score every query–doc pair jointly and can't be pre-indexed — *You can only afford it on a small shortlist; first-stage recall narrows millions to ~50 cheaply.*
- ▫️ Cross-encoders can't read long text — *You truncate to ~512 tokens, but the real blocker is per-query cost, not length.*

**Q2 · 🔴 recall@50 is broken — the gold chunk often isn't retrieved at all. A teammate adds a cross-encoder reranker to fix it. Will it work?**

- ✅ No — a reranker only re-orders the shortlist; it can't recover a chunk that first-stage retrieval never fetched — *Reranking fixes "in the set but low," not "not retrieved." Broken recall needs chunking/hybrid/ANN-param/ColBERT fixes first.*
- ▫️ Yes — cross-encoders are accurate enough to find anything — *A cross-encoder only scores candidates it's handed; if the gold chunk isn't in the shortlist, it's invisible.*
- ▫️ Yes, if you also raise top_n to 50 — *Keeping more of a shortlist that already excludes the gold chunk still can't surface it — the problem is upstream recall.*

**Q3 · 🟡 A grounded factual QA system over your own clean docs has good recall. A teammate proposes HyDE to "boost retrieval." Best call?**

- ▫️ Add HyDE everywhere — it always improves retrieval — *HyDE's hypothetical answer can hallucinate details that pull retrieval away from the true passage.*
- ✅ Skip HyDE here — it helps zero-shot/domain-mismatch corpora, but can backfire on well-grounded factual lookups — *HyDE shines on vocabulary mismatch; on a clean grounded corpus with good recall it adds an LLM call and risk for little gain.*
- ▫️ Replace retrieval with HyDE-only — *HyDE is a query transform feeding retrieval, not a retriever.*

**Q4 · 🟡 Users mostly ask narrow factual lookups ("what's the refund window for plan X?"). A teammate wants to switch the whole system to GraphRAG. Best call?**

- ▫️ Yes — GraphRAG is strictly better than vector RAG — *It isn't; on lookups vector RAG + rerank beats it on quality and cost, and GraphRAG indexing is 5–10× pricier.*
- ✅ No — keep vector RAG + rerank for lookups; reserve GraphRAG for whole-corpus sensemaking questions — *Match the technique to the query distribution.*
- ▫️ Add agentic multi-hop retrieval to every query instead — *That multiplies latency 4–7× for single-hop questions that don't need it.*

---

**Q5 · 🟡 An interviewer says "design a Q&A assistant over our internal docs." Strongest first move?**

- ▫️ Start drawing embed → retrieve → generate immediately to show you know the pipeline — *Jumping to a generic diagram skips the requirements that determine the architecture — it reads as memorized.*
- ✅ Ask scoping questions first — shape, scale, freshness SLA, tenancy/permissions, stakes — then design to those constraints — *"RAG" hides three different systems; clarifying is what lets you pick index, ingestion, and isolation deliberately.*
- ▫️ Recommend the latest models and a managed vector DB up front — *Tool choices don't answer the requirements and signal you're skipping the design reasoning.*

## Reported & representative open questions

1. **Why add re-ranking on top of vector retrieval? Cross-encoder vs bi-encoder?** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *separate vs joint encoding; recall vs precision are different jobs.*
2. **How do you do two-stage retrieval + reranking under latency pressure?** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *fetch ~50 cheap → rerank to 5; budget ~50–150ms; fewer candidates / smaller model / cache.*
3. **Describe HyDE and when you'd use it.** — ✅ Reported ([HyDE, Gao et al. arXiv 2212.10496](https://arxiv.org/abs/2212.10496)) — *hypothetical answer → embed → retrieve; helps vocabulary mismatch, hurts grounded factual.*
4. **Differentiate query rewriting, multi-query expansion, and step-back prompting.** — ✅ Reported ([RAG II: Query Transformations](https://medium.com/thedeephub/rag-ii-query-transformations-49865bb0528c)) — *clean vs paraphrase-and-fuse vs generalize-then-specialize.*
5. **How does query expansion / HyDE improve retrieval?** — ✅ Reported ([HyDE arXiv 2212.10496](https://arxiv.org/abs/2212.10496)) — *closes the query–document vocabulary gap.*
6. **When is GraphRAG or agentic retrieval worth its cost vs vanilla vector RAG?** — 🔮 Representative — *sensemaking / genuine multi-hop only; both are off-distribution for narrow lookups.*
