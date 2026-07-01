# Retrieval & Architecture — Questions

> *Part of the [RAG question bank](README.md) · pair with [content/03-retrieval-and-hybrid-search.md](../content/03-retrieval-and-hybrid-search.md)*

Legend: ✅ correct · ▫️ distractor (read the *why*) · 🟢 Core · 🟡 Senior · 🔴 Staff · **provenance:** ✅ Reported / 🔮 Representative

---

## Multiple-choice checkpoints

**Q1 · 🟢 You're building support QA over a fixed 40k-doc knowledge base with a 1.5s latency budget. Which "shape" framing should drive your architecture?**

- ▫️ Web-shaped — optimize a single engine for seconds-fresh, billion-doc search — *That's Perplexity's problem; a closed 40k-doc corpus doesn't need web-scale freshness machinery.*
- ✅ Chat-shaped — closed corpus, comfortable latency; spend effort on chunking, reranking, and model choice — *A bounded corpus with a 1.5s budget is the chat shape; the dominant levers are indexing quality and reranking, not the vector DB.*
- ▫️ It doesn't matter — all RAG is the same pipeline — *Shape sets your latency budget, footprint, and dominant lever; conflating them is how you over- or under-build.*

**Q2 · 🟢 In an interview you're asked "RAG or fine-tuning to make the model answer from our internal wiki?" Strongest opening?**

- ▫️ Fine-tune the model on the wiki so the facts live in the weights — *Fine-tuning bakes in facts that go stale on every edit, gives no citations, and can't enforce per-user permissions.*
- ✅ RAG — retrieve at query time for freshness, citations, and access control; consider fine-tuning only for style/format or a fixed skill — *RAG decouples knowledge from weights: re-index for freshness, cite the chunk, filter by ACL.*
- ▫️ Neither — just use a bigger base model — *A bigger model still has no row for your private, post-training data.*

**Q3 · 🟢 A user searches for the exact error code "E1042." Which retriever most reliably surfaces it?**

- ▫️ Dense vector search — *Embeddings blur rare exact tokens — "E1042" may not sit near anything useful in vector space.*
- ✅ Keyword / BM25, inside a hybrid (with a sane analyzer) — *Lexical search matches the exact token; hybrid keeps semantic recall for the rest — just make sure the analyzer doesn't strip the code.*
- ▫️ A bigger embedding model — *Still an embedding — exact rare tokens remain its weak spot regardless of size.*

**Q4 · 🟡 Why do production systems like Perplexity fuse BM25 + dense and then run a cross-encoder, instead of one bigger vector search?**

- ▫️ Because vector search is too slow at scale — *ANN is fast; the issue is quality, not speed — recall and precision are different jobs.*
- ✅ Recall and precision are different jobs: fuse cheap recall (BM25+dense), then spend a precise but costly reranker on the shortlist — *A wide cheap net maximizes the chance the right doc is present; the cross-encoder re-orders only the shortlist.*
- ▫️ To avoid using a vector database at all — *They still use vector retrieval — it's one stage, fused with lexical and reranked.*

**Q5 · 🟡 Hybrid is on, but searches for part numbers like "ACME-X200" still miss even though the docs contain them. Most likely cause?**

- ✅ The BM25 analyzer is splitting/stripping the hyphen so "ACME-X200" never becomes a matchable token — *BM25 only matches tokens the analyzer emits; a default analyzer that strips punctuation kills exact-code matching — the classic hybrid gotcha.*
- ▫️ RRF is weighting dense too heavily — *If BM25 never produced the token, no fusion weight can surface it — the failure is upstream in tokenization.*
- ▫️ The embedding model is too small — *Exact rare tokens are dense retrieval's weak spot regardless of size; this is a lexical-analyzer problem.*

**Q6 · 🔴 A teammate wants to replace RRF with tuned weighted fusion (α·bm25 + (1−α)·dense) to squeeze a few nDCG points. Senior caveat?**

- ▫️ Weighted fusion is always worse than RRF — *Not true — a calibrated α can beat RRF by a few points; the issue is cost, not a hard ceiling.*
- ✅ The weight is corpus/query-distribution-specific, so it goes stale on drift and needs a labelled set plus ongoing re-tuning — *RRF's zero-maintenance robustness is usually worth more than the small nDCG gain.*
- ▫️ Weighted fusion can't combine BM25 and cosine scales — *It can, via score normalization; the real problem is maintaining the weight over time.*

**Q7 · 🟡 Queries are pure natural-language paraphrase over a small, clean FAQ with no codes or proper nouns. A teammate insists on adding BM25 + RRF. Best call?**

- ▫️ Add it — hybrid is always better — *Hybrid's win comes from exact-token queries; with none, you add an index and latency for little gain.*
- ▫️ Add SPLADE instead, it's strictly superior — *SPLADE shines on vocabulary mismatch with exact-match needs; for pure paraphrase it's unnecessary complexity too.*
- ✅ Skip hybrid here — measure first; pure dense may already be enough for paraphrase-only traffic — *Hybrid isn't free; when the query distribution has no exact-token needs, it often buys nothing — let the metric decide.*

**Q8 · 🔴 An interviewer asks why you don't just stuff the whole corpus into a 1M-token context and skip retrieval. Strongest answer?**

- ▫️ Long context is cheaper than running a vector DB — *The opposite — you pay for every dumped token on every query.*
- ✅ It loses on cost (per-token), latency (prefill), recall (lost-in-the-middle), and attribution (no chunk to cite) — retrieve to keep the window small, cheap, ordered, and citable — *All four axes favour retrieval for corpus-scale QA; long context is a complement, not a replacement.*
- ▫️ Models can't actually read 1M tokens — *They can ingest it, but recall degrades with length and the cost/latency/attribution problems remain.*

**Q9 · 🔴 A regulated customer asks you to "deploy your RAG product into our VPC." Strongest framing of the actual deliverable?**

- ▫️ A vector database and an LLM endpoint running on their cloud account — *That's only the data plane; without identity, policy, audit, and guardrails you've shipped the demo.*
- ✅ A control plane (identity, policy, audit, guardrails) wrapped around a data plane (ingest, retrieve, generate), with the model as a swappable config choice — *The deliverable is accountability: every layer emits a signed, user-keyed artifact, and the model vendor can change without touching the SSO/RBAC/audit story.*
- ▫️ A fine-tuned model trained on the customer's documents — *Fine-tuning bakes in stale facts, gives no per-user permissions or citations.*

**Q10 · 🟡 You're scoping a deployment for a customer who may later move off a hosted model to self-hosted open-weights. What design choice protects that path best?**

- ▫️ Standardize on the hosted model's proprietary features so the integration is as tight as possible — *Tight coupling to one vendor's API is exactly what forces a re-platform later.*
- ✅ Treat the model as a swappable endpoint behind the gateway, so identity, policy, retrieval, and audit never depend on the model vendor — *If the control plane is model-agnostic, swapping Bedrock for Llama-on-EKS is a config change, not a rebuild.*
- ▫️ Avoid the question — model choice never changes once a deployment ships — *Klarna and others demonstrably re-balanced model choice post-launch under cost/quality pressure.*

---

## Reported & representative open questions

> Practice the **structure**, not a memorized script: clarify requirements → name the stage/metric → propose the cheapest lever → state the tradeoff.

1. **Explain the main parts of a RAG system and how they work.** — 🔮 Representative ([DataCamp](https://www.datacamp.com/blog/rag-interview-questions))
2. **What are the main benefits of RAG vs relying on the LLM's internal knowledge?** — ✅ Reported ([DataCamp](https://www.datacamp.com/blog/rag-interview-questions)) — *freshness, citations, access control, smaller model viable.*
3. **How do you choose the right retriever for a RAG app?** — ✅ Reported ([DataCamp](https://www.datacamp.com/blog/rag-interview-questions)) — *by query distribution: dense / BM25 / hybrid.*
4. **What is hybrid search — vector + keyword (BM25)?** — ✅ Reported ([DataCamp](https://www.datacamp.com/blog/rag-interview-questions), [Adil Shamim, 100 real interviews](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a))
5. **How do you ensure retrieved information is relevant and accurate?** — ✅ Reported ([DataCamp](https://www.datacamp.com/blog/rag-interview-questions)) — *recall@k/precision@k, reranking, grounding.*
6. **Scale a RAG system to 10M+ articles (sharding, caching, retrieval optimization).** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *see [08-production](../content/08-production-latency-cost.md).*
7. **Your RAG returns relevant docs but users can't find the answer — how do you turn a search engine into an answer engine?** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *grounding + citations + reranking to the right chunk.*
8. **How does ANN search work — walk through HNSW indexing.** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *see [02-embeddings & vector DBs](../content/02-embeddings-and-vector-dbs.md).*
9. **Design a Q&A assistant over our internal docs.** — ✅ Reported ([Oracle 2026, via HelloInterview](https://www.hellointerview.com/community/questions/ai-tutoring-platform/cm75445ag00053b64gz80nijf)) — *start with scoping questions; see the [rubric](../answers/rag-system-design-rubric.md).*
