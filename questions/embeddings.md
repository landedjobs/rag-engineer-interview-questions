# Embeddings & Vector Stores — Questions

> *Part of the [RAG question bank](README.md) · pair with [content/02-embeddings-and-vector-dbs.md](../content/02-embeddings-and-vector-dbs.md)*

Legend: ✅ correct · ▫️ distractor · 🟢 Core · 🟡 Senior · 🔴 Staff · ✅ Reported / 🔮 Representative

---

## Multiple-choice checkpoints

**Q1 · 🟢 A teammate proposes fixing weak answers by upgrading to the largest available embedding model. What's the senior response?**

- ▫️ Agree — bigger embeddings are the main driver of retrieval quality — *Model size is rarely the dominant lever; domain fit, chunking, and hybrid search move retrieval far more.*
- ✅ First measure recall@k / precision@k to locate the failure, then fix chunking/hybrid before swapping models — *Retrieval is an IR problem — diagnose with metrics; the cheapest wins are chunking + hybrid, not a bigger encoder.*
- ▫️ Switch to a larger generation model instead — *If the right chunk isn't retrieved, no generator can recover it — this is a retrieval problem.*

**Q2 · 🟡 recall@20 is stuck at 0.86 and tuning chunking hasn't moved it. You're on HNSW with default params. What's worth trying first?**

- ✅ Raise efSearch (and/or M) — ANN is approximate, so the index config may be the recall ceiling — *A low efSearch silently drops true top-k; raising it tests whether the index, not the encoder, caps recall — and it's free.*
- ▫️ Immediately swap to a larger embedding model — *A model swap means a full re-embed and is rarely the cheapest lever; rule out the approximate-search ceiling first.*
- ▫️ Lower k to 5 so precision improves — *Shrinking k can only lower recall — it removes candidates rather than recovering the missed gold chunk.*

**Q3 · 🔴 You're serving 50M chunks and the vector index won't fit your RAM budget. Which move shrinks the footprint WITHOUT re-embedding?**

- ▫️ Re-embed everything with a smaller model — *That's exactly the re-embed you're trying to avoid — and it invalidates every stored vector.*
- ▫️ Increase the chunk size so there are fewer vectors — *Bigger chunks dilute the embedding and hurt precision; a quality regression, not a clean memory fix.*
- ✅ Truncate Matryoshka dimensions and/or switch to product quantization (IVFPQ), recovering quality with a reranker — *Both shrink stored vectors in place at a measured recall cost a reranker restores — no re-embed.*

**Q4 · 🔴 You must serve 800M vectors but your RAM budget is tight. Which index choice fits — and what's the cost?**

- ▫️ HNSW — best recall, accept the RAM bill — *HNSW at ~1.4 TB for a billion vectors may simply not fit; "best recall" is moot if you can't afford to host it.*
- ✅ IVFPQ — ~20× less memory, then recover quality with a reranker on the shortlist — *IVFPQ trades recall for a huge memory saving (~70 GB vs ~1.4 TB at 1B); pairing with reranking restores top-k quality.*
- ▫️ It doesn't matter — all ANN indexes use similar memory — *They differ by ~20× at billion scale; index choice is a procurement decision.*

**Q5 · 🔴 You upgrade your embedding model and push the new vectors into the existing live index alongside the old ones. What happens?**

- ▫️ Recall improves immediately since the new model is better — *New and old vectors live in different spaces; comparing across them is meaningless, so quality degrades.*
- ▫️ Nothing — embeddings from different models are comparable — *They are not — a new model's vector space isn't cosine-comparable to the old one (representation shearing).*
- ✅ Recall shears silently because old and new vectors aren't comparable; full-reindex behind a dual index and cut over — *Mixing generations corrupts cross-comparisons; the safe path is re-embed in a parallel index, validate, then atomic cutover.*

**Q6 · 🟡 A 4-person startup is launching B2B RAG with ~2M vectors per customer and a strict data-residency requirement. Build or buy the vector store?**

- ▫️ Self-host Vespa from day one for maximum control — *A 4-person team can't absorb self-hosting at this scale; control isn't the binding constraint, residency is.*
- ✅ Buy a managed store, but pick one that supports per-tenant isolation / in-region (VPC) deployment to satisfy residency — *Below ~10M vectors managed is cheaper and faster; residency is satisfied by isolation/region features, not by self-hosting everything.*
- ▫️ Use one shared index with a metadata filter and ignore residency for now — *Residency is a legal constraint, not a "later" item, and a shared index risks cross-tenant leakage.*

---

## Reported & representative open questions

1. **What vector databases have you used? Which and why?** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *name the tradeoff axis, not a brand.*
2. **Do you need a vector database to implement RAG?** — ✅ Reported ([DataCamp](https://www.datacamp.com/blog/rag-interview-questions)) — *no, below a few thousand chunks; they earn their keep at scale + filtering + isolation.*
3. **Where do embeddings fail — negation, temporal reasoning, precision?** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *hence hybrid with BM25 for exact tokens.*
4. **How do you choose an embedding model?** — 🔮 Representative — *start from [MTEB](https://huggingface.co/spaces/mteb/leaderboard), filter by constraints, validate on your own corpus.*
5. **Estimate the index RAM for 50M chunks of 1,536-d embeddings on HNSW.** — 🔮 Representative — *N × dim × 4B ≈ 300 GB raw, ×~1.8 for HNSW ≈ 450–600 GB; then Matryoshka/PQ.*
6. **Why cosine and not Euclidean distance for embeddings?** — 🔮 Representative — *cosine is magnitude-invariant; on normalized vectors cosine/dot/L2 rank identically.*
