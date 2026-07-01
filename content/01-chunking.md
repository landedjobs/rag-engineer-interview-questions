# Chunking & Contextual Retrieval

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md) · drill the [chunking question bank](../questions/chunking.md)*

Chunking is the stage most candidates treat as a setting (`chunk_size=512`) and most seniors treat as an information-retrieval problem you *measure*. The interview tests tradeoff fluency: you size by a metric, you separate the match-unit from the read-unit, you know the orphaned-chunk failure and its two fixes, you treat parsing as its own stage, and you can change chunking in production without shearing recall. **Lead with the tension, then the lever and the number.**

---

## Parsing is stage zero

No chunker recovers a shredded parse. Scanned PDFs, multi-column layouts, and tables are corrupted *before* any splitter runs — interleaved columns become word-salad, table numbers detach from their headers, and every downstream stage inherits the garbage. If your corpus is scanned PDFs with tables and answers garble numeric facts, **look at the parser first**, not the chunker: use layout/OCR-aware extraction and keep tables as Markdown/HTML so structure survives into the chunk.

> [!WARNING]
> "My recall is bad, let me try a semantic chunker" on a corpus of scanned tables is tuning the wrong stage. Semantic chunking cannot recover structure the parser already destroyed. Parse → chunk → embed is a pipeline; garbage at stage zero is garbage at every stage.

---

## The five chunking families — and how each one fails

| Strategy | Unit | Fails when |
|---|---|---|
| **Fixed-size** (e.g. 512 tokens) | token count | cuts mid-sentence / mid-function; ignores document structure |
| **Recursive** (split on `\n\n`, `\n`, `. `) | text separators | good default, but blind to semantics and tables |
| **Structure-aware** (headings, functions via AST, clauses) | the document's own units | needs a parser per format; but keeps meaning intact |
| **Semantic** (embed sentences, split on similarity drop) | topic boundary | expensive, often *not worth it* vs recursive + good size |
| **Late / contextual** | (see below) | fixes the orphaned-chunk problem specifically |

The unit of meaning is rarely a fixed token count. Code's unit is the function/class (split on AST boundaries, attach file path + enclosing symbol as metadata — the Cursor approach). Contracts split on clauses; API docs on headings. **Split on the document's own units, carry the unit's identity as metadata, not a token ruler.**

---

## Size & overlap: stop guessing, measure `precision@5`

Chunk size is a tradeoff curve, not a constant. Bigger chunks carry more context but *dilute the embedding* — one pooled vector spreads the relevant signal across irrelevant tokens, similarity falls, the chunk ranks lower. Smaller chunks are precise but can strip the context a claim needs. The senior workflow: start ~512 with structure-aware splitting, then **sweep sizes against a labelled set** and read `recall@20` (did we fetch it?) vs `precision@5` (is the top-5 clean?).

```python
# Don't ship a chunking change on vibes -- sweep candidates against a labelled set.
def pick_chunk_size(eval_set, build_index, retrieve, sizes=(256, 384, 512, 768)):
    results = {}
    for size in sizes:
        build_index(chunk_size=size, overlap=int(size * 0.12))   # rebuild for each
        r20 = recall_at_k(eval_set, retrieve, k=20)   # did we fetch it?
        p5 = precision_at_k(eval_set, retrieve, k=5)  # is top-5 clean?
        results[size] = (round(r20, 3), round(p5, 3))
    return results
# Read it as a tradeoff curve: pick the size that holds recall@20 while maximizing
# precision@5. If the best p5 is still < 0.7, that's your signal to add a reranker.
```

> [!TIP]
> If `recall@20` is healthy but `precision@5` is stuck below ~0.7, don't keep shrinking chunks — that's the signal to add a **reranker** ([04](04-reranking-and-query-rewriting.md)). Chunk size sets the ceiling; the reranker buys the last mile of precision.

---

## The orphaned-chunk problem and its two fixes

A chunk that reads *"revenue grew 3% over the previous quarter"* loses which company and which quarter the moment it's isolated from its document — and it won't rank for "ACME Q2 2023 revenue" because those identifiers aren't in the text. This is the **orphaned-chunk failure**, and it's the most-asked chunking interview question. Two fixes:

**1. Contextual retrieval (Anthropic, Sep 2024).** Prepend an LLM-written context to each chunk *before* indexing it (into both embeddings and BM25). Prompt-cache the whole document once and reuse it per chunk, so the pass is ~90% cheaper. Measured impact: **~49% fewer failed retrievals, ~67% combined with a reranker.**

```python
# Contextual retrieval at index time: situate each chunk in its document.
# Prompt-cache the WHOLE document once, then reuse it for every chunk -> ~90% cheaper.
CONTEXT_PROMPT = (
    "Here is the whole document:\n{doc}\n\n"
    "Here is a chunk we want to situate within it:\n{chunk}\n\n"
    "Give a short (50-100 token) context that situates this chunk in the document "
    "for search -- resolve names, dates, and references. Answer only with the context."
)

def contextualize(doc, chunks, llm):
    out = []
    for c in chunks:
        ctx = llm(CONTEXT_PROMPT.format(doc=doc, chunk=c), cache_prefix=doc)  # doc cached
        out.append(ctx + "\n\n" + c)   # prepend, then index in BOTH embeddings + BM25
    return out
# Idempotent: re-run on a document UPDATE, or old and new chunk reps mix and shear recall.
```

**2. Late chunking (Jina).** Embed the *whole document* in one long-context forward pass, *then* split the token embeddings into chunks — so each chunk vector already carries document-wide context. Cheaper (one pass, no extra LLM call) but needs coherent documents and doesn't feed BM25.

**Interview framing:** contextual = extra LLM call, model-agnostic, also improves lexical search; late chunking = one forward pass, cheap, needs coherent docs. Name the tradeoff, don't pick blindly.

---

## Small-to-big: match unit ≠ read unit

You want precision at match time (small chunks) *and* enough context to answer (large chunks). Decouple them: index ~256-token **children** for matching, but return the ~1024-token **parent** to read, de-duped by parent id.

```python
# index children (small) -> retrieve children -> return de-duped parents
def retrieve_small_to_big(query, k_children=20, k_parents=5):
    children = vector_search(query, k=k_children)   # 256-token units
    best_by_parent = {}
    for c in children:
        p = c["parent_id"]
        if p not in best_by_parent or c["score"] > best_by_parent[p]["score"]:
            best_by_parent[p] = c   # dedupe: keep top child per parent
    ranked = sorted(best_by_parent.values(), key=lambda c: c["score"], reverse=True)
    return [load_parent(c["parent_id"]) for c in ranked[:k_parents]]  # ~1024-token context
```

---

## Changing chunking on a live index

Re-chunking is a full re-embed. **Never re-chunk in place** — old and new chunk representations mix in the same index and recall shears silently while every dashboard stays green. Build a second index with the new chunker, validate on a held-out labelled set, then cut traffic over atomically (and roll back instantly if it regresses).

> [!WARNING]
> **Senior trap:** "I'd use 512-token chunks" with no "…then I'd measure `precision@5` and adjust" tells an interviewer you treat chunking as a setting, not an IR problem. Same red flag: "bigger chunks for more context" (backwards — it dilutes) and "I'd re-chunk the live index in place" (shears recall). Always pair a starting point with a metric and a safe rollout.

---

## Interview angles (say these out loud)

- **"What chunk size and why?"** → start ~512 with structure-aware splitting, then measure `precision@5` and drop toward ~256 if it sags — never a fixed answer.
- **"Why don't bigger chunks just give more context?"** → one pooled vector dilutes the relevant signal across irrelevant tokens; similarity falls, the chunk ranks lower.
- **"A chunk says 'revenue grew 3%' with no company/quarter — fix?"** → contextual retrieval; ~49%/~67% fewer failures.
- **"Late chunking vs contextual retrieval?"** → late = one long-context pass, cheap, coherent docs; contextual = extra LLM call, model-agnostic, feeds BM25.
- **"How do you get precision AND context?"** → small-to-big: match on ~256-token children, return the ~1024-token parent, dedupe by parent id.
- **"How do you chunk code / contracts / API docs?"** → split on the document's own units (AST functions, clauses, headings); carry identity as metadata.
- **"Change chunking on a live index — how?"** → dual index, validate on a held-out set, atomic cutover.

---

## 📚 Resources

- 🧑‍🏫 [Contextual Retrieval](https://www.anthropic.com/engineering/contextual-retrieval) — Anthropic (Sep 2024) · the canonical orphaned-chunk fix; the ~49%/~67% numbers everyone cites.
- 📘 [Chunking Strategies for LLM Applications](https://www.pinecone.io/learn/chunking-strategies/) — Pinecone · docs · the standard tour of fixed/recursive/semantic.
- 📄 [Best Chunking Strategies for RAG 2026](https://www.firecrawl.dev/blog/best-chunking-strategies-rag) — Firecrawl (Feb 2026) · benchmark of 7 strategies (recursive, semantic, late, contextual).
- 📘 [Chunking Strategies for RAG](https://weaviate.io/blog/chunking-strategies-for-rag) — Weaviate (Sep 2025) · docs · practical with code.
- 📄 [Late Chunking vs Contextual Retrieval — the math](https://medium.com/kx-systems/late-chunking-vs-contextual-retrieval-the-math-behind-rags-context-problem-d5a26b9bbd38) — kx-systems · article · the tradeoff, quantified.
- 📄 [Reconstructing Context (Jina late chunking)](https://arxiv.org/html/2504.19754v1) — arXiv · paper · the late-chunking method in depth.
- 💻 [NirDiamant/RAG_Techniques](https://github.com/NirDiamant/RAG_Techniques) — **28k★, Apache-2.0** · notebooks for semantic chunking, contextual retrieval, small-to-big. *Link + study; adapt with attribution.*
