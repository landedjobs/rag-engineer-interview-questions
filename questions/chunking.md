# Chunking — Questions

> *Part of the [RAG question bank](README.md) · pair with [content/01-chunking.md](../content/01-chunking.md)*

Legend: ✅ correct · ▫️ distractor · 🟢 Core · 🟡 Senior · 🔴 Staff · ✅ Reported / 🔮 Representative

---

## Multiple-choice checkpoints

**Q1 · 🟡 A retrieved chunk says "revenue grew 3% over the previous quarter," but the model can't tell which company or quarter — and the right chunk often doesn't rank for "ACME Q2 2023 revenue." Strongest fix?**

- ▫️ Use much larger chunks so each carries more surrounding text — *Helps a little but dilutes the embedding and bloats the index — and still doesn't reliably inject the missing identifiers.*
- ✅ Contextual retrieval: prepend an LLM-written context to each chunk before indexing (embeddings + BM25) — *Situates every chunk in its document so identifiers survive isolation; Anthropic measured ~35–49% fewer failures, ~67% with a reranker.*
- ▫️ Switch to a larger generation model — *The identifiers were never retrieved — no generator can recover information that isn't in the context.*

**Q2 · 🟡 Your RAG has healthy recall@20, but answers are vague and miss specifics; precision@5 measures 0.55. Best first move?**

- ▫️ Increase chunk size to give each chunk more context — *Larger chunks usually lower precision — more noise per vector — the opposite of what you need.*
- ✅ Shrink chunks toward ~256 tokens and add a reranker to sharpen the top-5 — *precision@5 is the bottleneck: smaller chunks + a cross-encoder tighten exactly what reaches the prompt without losing recall.*
- ▫️ Swap the embedding model for a larger one — *Rarely the dominant lever; measure chunk size and reranking first.*

**Q3 · 🔴 You need to change your chunk size and contextualization prompt on a live 1 TB index. What's the safe rollout?**

- ▫️ Re-chunk and re-embed in place, one document at a time, during low traffic — *In-place re-embedding mixes old and new representations; recall shears silently while dashboards stay green.*
- ✅ Build a second index with the new chunker, validate on a held-out labelled set, then cut traffic over atomically — *A dual index keeps generations from mixing; switch only once the new one validates, roll back instantly.*
- ▫️ Keep both chunkers' vectors in one index and let RRF sort it out — *Cross-generation vectors aren't comparable; fusing them hides the sheared distribution, doesn't fix it.*

**Q4 · 🟡 Indexing a codebase, answers to "where is parseConfig defined?" keep missing or returning fragments. Your splitter is fixed-size 512 tokens. Best fix?**

- ✅ Split on language-aware boundaries (functions/classes via AST), attaching file path + enclosing symbol as metadata — *Code's unit of meaning is the function/class; AST splitting keeps definitions whole and the metadata makes "where defined" answerable — the Cursor approach.*
- ▫️ Increase the chunk size to 2,048 tokens so more code fits — *Bigger chunks dilute the embedding and still cut at arbitrary points; the issue is the wrong boundary, not size.*
- ▫️ Add overlap so fragments are duplicated across chunks — *Overlap patches straddling sentences, not a function split off from its signature/imports.*

**Q5 · 🟡 Your corpus is mostly scanned PDFs with tables. Recall is poor and answers garble numeric facts. Where do you look first?**

- ▫️ Swap to a semantic chunker — *Semantic chunking can't recover structure the parser already destroyed; wrong stage.*
- ▫️ Raise k and add a reranker — *More candidates can't fix chunks that are interleaved-column noise; the text was corrupted before indexing.*
- ✅ The parsing stage — use layout/OCR-aware extraction and keep tables as Markdown/HTML before chunking — *No chunker recovers a shredded parse; parsing is stage zero.*

---

## Reported & representative open questions

1. **What is late chunking vs traditional chunking?** — ✅ Reported ([DataCamp](https://www.datacamp.com/blog/rag-interview-questions)) — *embed the whole doc first, split the token embeddings second, so each chunk carries doc-wide context.*
2. **A financial report's page 1 says "all amounts in thousands" — how do you handle doc-wide context when chunking page-by-page?** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *contextual retrieval / late chunking; the orphaned-chunk problem.*
3. **Why is chunking important, and how do you choose chunk size?** — ✅ Reported ([KalyanKS-NLP](https://github.com/KalyanKS-NLP/RAG-Interview-Questions-and-Answers-Hub)) — *measure precision@5; start ~512, drop toward ~256 if it sags — never a fixed number.*
4. **What techniques handle very long documents or large knowledge bases in RAG?** — ✅ Reported ([DataCamp](https://www.datacamp.com/blog/rag-interview-questions)) — *structure-aware chunking, small-to-big, contextual retrieval, hierarchical summaries.*
5. **How do you chunk code / contracts / API docs differently from prose?** — 🔮 Representative — *split on the document's own units (AST, clauses, headings); carry identity as metadata.*
