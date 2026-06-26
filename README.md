# RAG Engineer Interview Questions

> Focused interview prep for **RAG (Retrieval-Augmented Generation) Engineer** roles — the questions that actually come up.
> Maintained by [Landed](https://landed.b100x.ai) — get **referred**, prep with mock interviews, and land the role.

⭐ **Star this** and contribute questions via PR.

---

## What a RAG Engineer does (and why it's hot)

A **RAG Engineer** connects LLMs to trusted data so answers are accurate, current, and grounded — retrieval pipelines, embeddings, vector stores, re-ranking, and evals. It's one of the fastest-growing AI-native specialties because every company wants AI that's right, not just fluent.

---

## Questions by topic

### Retrieval fundamentals
- What is RAG and when do you use it instead of fine-tuning?
- Walk through the full pipeline: ingestion → chunking → embedding → storage → retrieval → generation.
- How do you choose an embedding model? What trade-offs matter?
- Dense vs sparse vs hybrid retrieval — when each?

### Chunking & indexing
- How do you chunk documents, and why does it matter?
- How do metadata and filters improve retrieval?
- How do you handle tables, PDFs, and code in a knowledge base?

### Quality & accuracy
- How do you measure retrieval quality? (recall@k, MRR, nDCG)
- How do you reduce hallucinations in a RAG system?
- What is re-ranking and when is it worth the latency?
- How do you add citations / source attribution?

### Production & scale
- Design a RAG system over 10M+ documents — latency, cost, caching, freshness.
- How do you keep the index fresh as source data changes?
- How do you handle multi-tenant data isolation?
- How do you monitor a RAG system in production?

### Evals
- How would you build an eval set for a RAG app?
- Pros/cons of LLM-as-judge; how do you keep it honest?

### Coding
- Implement a minimal retrieve-then-answer endpoint with citations.
- Given messy retrieval results, write re-ranking / dedup logic.

---

## Strong-answer cheatsheet

- Always tie choices to **evals** — "I'd measure recall@k before and after."
- Mention **grounding + citations** for trust.
- Talk **cost/latency trade-offs**, not just accuracy.
- Have one **real project** you can whiteboard end-to-end.

---

## Practice

Run a **RAG Engineer mock interview** with role-specific feedback on [Landed](https://landed.b100x.ai), and find which companies are hiring RAG Engineers in the [job lists](https://github.com/landedjobs/awesome-ai-native-jobs).

---

<sub>Maintained by [Landed](https://landed.b100x.ai). PRs welcome — add questions you were asked.</sub>
