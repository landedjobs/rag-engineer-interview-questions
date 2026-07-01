# RAG Engineer Resources (2026)

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md)*

The curated resource hub — **50 typed, annotated, license-noted resources** across 16 categories, plus the **13 canonical repos to mine**. Every link is annotated (never a bare dump), typed, and — for repos — license-noted.

**Type legend:** 📄 paper · 📘 docs · 🎬 video · 🧑‍🏫 course/guide · 🛠️ tool · 💻 repo

> [!NOTE]
> **License discipline:** this repo ships MIT. We adapt only MIT / Apache-2.0 / CC0 code (with attribution); CC-BY-SA and research-artifact-licensed sources (e.g. autogen CC-BY-4.0) are **link + commentary only, never copied verbatim**. Repo stars/licenses are as of mid-2026 — verify before reuse.

---

## 1. Foundational RAG repos

- 💻 [NirDiamant/RAG_Techniques](https://github.com/NirDiamant/RAG_Techniques) — **28k+★, Apache-2.0** · 30+ notebooks: semantic chunking, HyDE, GraphRAG, RAGAS eval, contextual retrieval, agentic RAG. The best single hands-on reference.
- 💻 [microsoft/graphrag](https://github.com/microsoft/graphrag) — **34k★, MIT** · modular graph-based RAG; canonical graph-shaped retrieval.
- 💻 [run-llama/llama_index](https://github.com/run-llama/llama_index) — **50.5k★, MIT** · leading RAG framework; base for many agentic RAG demos.
- 💻 [Danielskry/Awesome-RAG](https://github.com/Danielskry/Awesome-RAG) — Apache-2.0 · curated index of RAG papers/tutorials — the discovery layer.

## 2. RAG interview-question repos

- 💻 [KalyanKS-NLP/RAG-Interview-Questions-and-Answers-Hub](https://github.com/KalyanKS-NLP/RAG-Interview-Questions-and-Answers-Hub) — 100+ RAG Qs w/ answers, topically organized. The most directly-named RAG interview repo.
- 💻 [llmgenai/LLMInterviewQuestions](https://github.com/llmgenai/LLMInterviewQuestions) — **1.8k★** · 100+ LLM/RAG Qs from Google/NVIDIA/Meta/Microsoft/Fortune 500.
- 💻 [a-tabaza/genai_interview_questions](https://github.com/a-tabaza/genai_interview_questions) — **76★** · theory/fundamentals Q&A.

## 3. Chunking

- 📄 [Best Chunking Strategies for RAG 2026](https://www.firecrawl.dev/blog/best-chunking-strategies-rag) — Firecrawl (Feb 2026) · benchmark of 7 strategies.
- 📘 [Chunking Strategies for LLM Applications](https://www.pinecone.io/learn/chunking-strategies/) — Pinecone · the standard tour.
- 📘 [Chunking Strategies for RAG](https://weaviate.io/blog/chunking-strategies-for-rag) — Weaviate (Sep 2025) · practical with code.
- 📄 [Late Chunking vs Contextual Retrieval — the math](https://medium.com/kx-systems/late-chunking-vs-contextual-retrieval-the-math-behind-rags-context-problem-d5a26b9bbd38) — kx-systems · the tradeoff, quantified.
- 📄 [Reconstructing Context (Jina late chunking)](https://arxiv.org/html/2504.19754v1) — arXiv · the late-chunking method.

## 4. Embeddings & selection

- 🛠️ [MTEB Leaderboard](https://huggingface.co/spaces/mteb/leaderboard) — HuggingFace · the canonical embedding selector.
- 📄 [Best Embedding Model for RAG 2026: 10 Models](https://milvus.io/blog/choose-embedding-model-rag-2026.md) — Milvus (Mar 2026) · Gemini Embedding 2 best all-rounder.
- 📄 [5 Best Embedding Models for RAG](https://greennode.ai/blog/best-embedding-models-for-rag) — GreenNode (Oct 2025).
- 📄 [Long-Context Retrieval with Monarch Mixer](https://hazyresearch.stanford.edu/blog/2024-01-11-m2-bert-retrieval) — Stanford Hazy · the long-context embedding frontier.

## 5. Vector databases

- 📄 [Best Vector Databases 2026](https://www.firecrawl.dev/blog/best-vector-databases) — Firecrawl (May 2026).
- 📄 [Pinecone vs Weaviate vs Qdrant vs pgvector](https://openhelm.ai/blog/pinecone-vs-weaviate-vs-qdrant-vs-pgvector) — openhelm.ai (Sep 2025).
- 📄 [Pinecone vs Weaviate vs Qdrant vs Milvus](https://tensorblue.com/blog/vector-database-comparison-pinecone-weaviate-qdrant-milvus-2025) — tensorblue · the latency/QPS table.
- 📄 [10 Best Vector DBs for RAG](https://www.zenml.io/blog/vector-databases-for-rag) — ZenML (Oct 2025) · procurement angle.

## 6. Hybrid search (BM25 + dense)

- 📘 [Hybrid Search Explained](https://weaviate.io/blog/hybrid-search-explained) — Weaviate (Jan 2025) · the RRF walkthrough.
- 📄 [Hybrid Search in RAG: Dense + Sparse + BM25 + SPLADE + RRF](https://blog.gopenai.com/hybrid-search-in-rag-dense-sparse-bm25-splade-reciprocal-rank-fusion-and-when-to-use-which-fafe4fd6156e) — gopenai (Mar 2026).
- 📄 [From BM25 to Corrective RAG: Benchmarking Retrieval](https://arxiv.org/html/2604.01733v1) — arXiv (Apr 2026) · BM25 beats `text-embedding-3-large` except Recall@20.
- 📘 [Hybrid search docs](https://docs.weaviate.io/weaviate/search/hybrid) — Weaviate · API reference.

## 7. Re-ranking & late interaction

- 📘 [Rerankers and Two-Stage Retrieval](https://www.pinecone.io/learn/series/rag/rerankers/) — Pinecone · the canonical explainer.
- 🛠️ [Cohere Rerank](https://cohere.com/rerank) — Cohere · the managed reranker most teams reach for.
- 📄 [How Cohere Rerank-4 Improves RAG](https://orq.ai/blog/from-noise-to-signal-how-cohere-rerank-4-improves-rag) — orq.ai (Dec 2025).
- 📄 [Using LLMs as a Reranker for RAG](https://fin.ai/research/using-llms-as-a-reranker-for-rag-a-practical-guide/) — Fin (Sep 2025).
- 📄 [Flexible Training/Retrieval for Late Interaction (ColBERT/Jina)](https://arxiv.org/html/2508.03555v1) — arXiv (Aug 2025).

## 8. Query rewriting & HyDE

- 📄 [HyDE: Precise Zero-Shot Dense Retrieval without Relevance Labels](https://arxiv.org/abs/2212.10496) — Gao et al. · the original HyDE paper.
- 📘 [HyDE (Haystack docs)](https://docs.haystack.deepset.ai/docs/hypothetical-document-embeddings-hyde) — deepset · implementation reference.
- 📄 [RAG II: Query Transformations](https://medium.com/thedeephub/rag-ii-query-transformations-49865bb0528c) — The Deep Hub · rewriting vs multi-query vs step-back.
- 📄 [Build Advanced RAG: Query Rewriting](https://dev.to/rogiia/build-an-advanced-rag-app-query-rewriting-h3p) — rogii · hands-on.
- 📘 [MultiQueryRetriever](https://reference.langchain.com/python/langchain-classic/retrievers/multi_query/MultiQueryRetriever) — LangChain · the multi-query pattern.

## 9. Retrieval evaluation

- 📄 [Evaluation Metrics for Search / Rec Systems](https://weaviate.io/blog/retrieval-evaluation-metrics) — Weaviate · precision, recall, MRR, MAP, nDCG.
- 📘 [Ragas: Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) · [Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) — the metric definitions.
- 📄 [Contextual precision vs recall](https://milvus.io/ai-quick-reference/how-do-metrics-like-contextual-precision-and-contextual-recall-help-in-rag-evaluation) — Milvus.
- 📄 [Evaluating RAG with RAGAs](https://www.vectara.com/blog/evaluating-rag) — Vectara (Apr 2024).
- 📄 [Contextual AI Platform Benchmarks 2025](https://contextual.ai/blog/platform-benchmarks-2025/) — Contextual AI · BEIR 61.2 vs Voyage-v2 58.3.

## 10. RAG eval frameworks

- 💻 [confident-ai/deepeval](https://github.com/confident-ai/deepeval) — **16.6k★, Apache-2.0** · 15+ benchmarks; agentic/chatbot/RAG metrics.
- 💻 [explodinggradients/ragas](https://github.com/explodinggradients/ragas) — **14.6k★, Apache-2.0** · RAG-specific: faithfulness, context precision/recall.
- 💻 [truera/trulens](https://github.com/truera/trulens) — **3.4k★, MIT** · eval + tracking; groundtruth-based.
- 📄 [DeepEval vs Ragas](https://deepeval.com/blog/deepeval-vs-ragas) · [DeepEval vs TruLens](https://deepeval.com/blog/deepeval-vs-trulens) · [RAG Eval Tools roundup](https://aimultiple.com/rag-evaluation-tools) — AImultiple (Mar 2026).

## 11. Advanced RAG: contextual, GraphRAG, agentic

- 🧑‍🏫 [Contextual Retrieval in AI Systems](https://www.anthropic.com/engineering/contextual-retrieval) — Anthropic (Sep 2024) · the ~49%/~67% technique.
- 📘 [Welcome to GraphRAG](https://microsoft.github.io/graphrag/) — Microsoft · docs + methodology.
- 📄 [Agentic RAG Patterns 2026](https://www.digitalapplied.com/blog/agentic-rag-patterns-multi-step-reasoning-guide) — digitalapplied (Apr 2026).
- 📄 [Next-Gen Agentic RAG with LangGraph (2026)](https://medium.com/@vinodkrane/next-generation-agentic-rag-with-langgraph-2026-edition-d1c4c068d2b8).
- 📄 [What is Agentic RAG? 2026](https://www.lyzr.ai/blog/agentic-rag/) — Lyzr.
- 📄 [Self-RAG](https://arxiv.org/abs/2310.11511) · [CRAG](https://arxiv.org/abs/2401.15884) — arXiv · the two foundational agentic-RAG papers.
- 📄 [From RAG to GraphRAG](https://www.alexanderthamm.com/en/blog/from-rag-to-graphrag/) — Alexander Thamm (Jul 2025).

## 12. Grounding, citations, hallucination

- 📄 [Hallucination Mitigation for RAG-LLMs](https://www.mdpi.com/2227-7390/13/5/856) — MDPI (2025) · the mitigation taxonomy.
- 📄 [CiteFix: Post-hoc Citation](https://arxiv.org/html/2504.15629v2) — arXiv.
- 📄 [Generation-Time vs Post-hoc Citation (REASONS)](https://arxiv.org/html/2509.21557v2) — arXiv (Sep 2025).
- 🎬 [Unlocking Advanced RAG: Citations & Attributions](https://www.youtube.com/watch?v=RnCuOL-LBAw) — video.

## 13. Enterprise / secure / multi-tenant RAG

- 📄 [Multi-Tenant RAG with pgvector + RLS](https://medium.com/@rockingmanas78/how-to-create-a-multi-tenant-rag-44aa0fefa383).
- 📄 [Building Multi-Tenant RAG with PostgreSQL](https://www.tigerdata.com/blog/building-multi-tenant-rag-applications-with-postgresql-choosing-the-right-approach) — TigerData · the decision tree.
- 📄 [Multi-Tenant RAG: RLS in pgvector with MCP](https://blog.techtush.in/multi-tenant-rag-row-level-security-in-pgvector-with-mcp) — techtush (May 2026).
- 💻 [microsoft/presidio](https://github.com/microsoft/presidio) — **MIT** · PII detect/redact/mask/anonymize.
- 📄 [Access Control Secrets for RAG](https://www.lmdconsulting.com/blogs/unlocking-secure-ai-access-control-secrets-for-rag-systems) — LMD Consulting (May 2025) · RBAC + ABAC.
- 📄 [Secure LLMs, RAG & Agentic AI](https://www.enkryptai.com/blog/enterprise-ai-security-framework-2025-securing-llms-rag-and-agentic-ai) — Enkrypt AI (Jun 2025).
- 📄 [Confidential RAG Pipeline (Intel TDX)](https://openmetal.io/resources/blog/how-to-build-a-confidential-rag-pipeline-that-guarantees-data-privacy/) — OpenMetal (Jan 2026).
- 📘 [OWASP LLM Top 10](https://genai.owasp.org/llm-top-10/) · [Indirect Prompt Injection](https://www.lakera.ai/blog/indirect-prompt-injection) — Lakera (CVE-2025-59944).

## 14. Interview loop guides & question lists

- 📄 [Top 30 RAG Interview Questions 2026](https://www.datacamp.com/blog/rag-interview-questions) — DataCamp · the most-cited 2026 list.
- 📄 [Every AI Engineer Interview Question 2026 (100+ real)](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a) — Adil Shamim (May 2026).
- 📄 [AI Engineer Interview Questions 2026](https://www.kore1.com/ai-engineer-interview-questions-2026/) — KORE1 · the loop structure.
- 📄 [How to Crack the AI Engineer Interview 2026](https://aiengineeringinsider.substack.com/p/how-to-crack-the-ai-engineer-interview) — Substack.
- 📄 [CareerY AI Engineer Q&A](https://careery.pro/blog/ai-careers/ai-engineer-interview-questions) — CareerY (Feb 2026).

## 15. Courses

- 🧑‍🏫 [Retrieval-Augmented Generation (RAG)](https://www.deeplearning.ai/courses/retrieval-augmented-generation) — DeepLearning.AI.
- 🧑‍🏫 [LangChain: Chat with Your Data](https://www.deeplearning.ai/courses/langchain-chat-with-your-data) — DeepLearning.AI (Andrew Ng).
- 🧑‍🏫 [RAG with LlamaIndex & LangChain](https://learn.activeloop.ai/courses/rag) — Activeloop.

## 16. Real-interview threads

- 🎬 [Got grilled in an ML interview (LangGraph)](https://www.reddit.com/r/LangChain/comments/1k662xc/got_grilled_in_an_ml_interview_today_for_my/) — Reddit 2025.
- 🎬 [Design AI tutoring platform w/ RAG (Oracle 2026)](https://www.hellointerview.com/community/questions/ai-tutoring-platform/cm75445ag00053b64gz80nijf) — HelloInterview.
- 🎬 [Interview prep (r/LLMDevs)](https://www.reddit.com/r/LLMDevs/comments/1piahou/interview_prep/) — Reddit.

---

## The 13 canonical repos to mine (stars + license)

| Repo | Stars | License | Mine it for |
|---|---|---|---|
| [NirDiamant/RAG_Techniques](https://github.com/NirDiamant/RAG_Techniques) | 28k+ | Apache-2.0 | technique notebooks (chunking → agentic) |
| [microsoft/graphrag](https://github.com/microsoft/graphrag) | 34k+ | MIT | graph-shaped retrieval |
| [run-llama/llama_index](https://github.com/run-llama/llama_index) | 50.5k | MIT | retrievers, fusion, query engines |
| [Danielskry/Awesome-RAG](https://github.com/Danielskry/Awesome-RAG) | — | Apache-2.0 | discovery / paper index |
| [explodinggradients/ragas](https://github.com/explodinggradients/ragas) | 14.6k | Apache-2.0 | RAG eval metrics |
| [confident-ai/deepeval](https://github.com/confident-ai/deepeval) | 16.6k | Apache-2.0 | benchmark-driven eval, CI |
| [truera/trulens](https://github.com/truera/trulens) | 3.4k | MIT | eval + tracking |
| [KalyanKS-NLP/RAG-Interview-Hub](https://github.com/KalyanKS-NLP/RAG-Interview-Questions-and-Answers-Hub) | — | likely MIT | question ideas (verify license) |
| [llmgenai/LLMInterviewQuestions](https://github.com/llmgenai/LLMInterviewQuestions) | 1.8k | see repo | company-sourced Qs |
| [a-tabaza/genai_interview_questions](https://github.com/a-tabaza/genai_interview_questions) | 76 | see repo | fundamentals Q&A |
| [zilliztech/GPTCache](https://github.com/zilliztech/gptcache) | — | Apache-2.0 | semantic caching reference |
| [microsoft/presidio](https://github.com/microsoft/presidio) | — | MIT | PII redaction |
| [langfuse/langfuse](https://github.com/langfuse/langfuse) | 30.1k | MIT | self-hostable observability spine |

> Stars/licenses are approximate as of mid-2026. Always check the repo's `LICENSE` before reusing code; adapt only MIT/Apache/CC0 with attribution.
