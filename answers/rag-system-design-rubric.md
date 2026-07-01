# The RAG System-Design Rubric

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md)*

One reusable rubric, applied across every worked design in [answers/](.). A 45–60 minute RAG system-design round almost always opens with *"design a Q&A assistant over our internal docs."* The difference between a mid and a senior signal is **structure**: clarify requirements, sketch the pipeline in three swim-lanes, defend each stage with a metric and a tradeoff, and name the silent failures unprompted.

---

## Step 0 — Clarify before you draw (the single best opening move)

The same word "RAG" hides three different systems. **Ask these five before drawing anything** — each answer collapses a design branch:

| Question | What it sets |
|---|---|
| **Shape?** chat over a closed corpus / enterprise search / web-scale | latency budget + dominant lever |
| **Scale?** how many docs/chunks, what QPS | index choice: managed HNSW vs IVFPQ + rerank |
| **Freshness?** how stale can an answer be | ingestion: nightly batch vs CDC streaming |
| **Tenancy?** single org or multi-tenant with per-user permissions | isolation architecture + the ACL filter |
| **Stakes?** is a wrong answer embarrassing or dangerous | eval rigor, injection guards, human-in-the-loop |

> Jumping straight to `embed → retrieve → generate` reads as memorized, not designed. Clarifying first is the senior tell.

---

## The three swim-lanes

Narrate the design as three separate concerns — quality, latency, safety:

```
OFFLINE INGEST (sets quality)          ONLINE QUERY (sets latency)        DEPLOY GATE (makes it safe)
  parse (layout/OCR-aware)               query rewrite (if needed)          split eval on prod-derived golden set
  chunk (structure-aware) + contextualize  filtered hybrid retrieve         injection tests on retrieved channel
  attach metadata {doc_id, acl, mtime}     rerank to ~5                     p99 latency + cost budget check
  embed + index (HNSW / IVFPQ + BM25)      ground + cite per claim          pinned model/index versions
```

Candidates who only draw the query path miss half the system — and all the parts that break in production.

---

## The rubric — score each stage on mechanism + metric + tradeoff

| # | Stage | Must name | Metric | Common trap |
|---|---|---|---|---|
| 1 | **Problem framing** | shape / scale / freshness / tenancy / stakes | — | drawing before clarifying |
| 2 | **Parsing** | layout/OCR-aware; tables → Markdown | — | "it's just text extraction" |
| 3 | **Chunking** | structure-aware + contextual; size by metric | `precision@5` | fixed 512 with no measurement |
| 4 | **Embedding + index** | domain-fit model; HNSW vs IVFPQ; RAM arithmetic | `recall@k` | "bigger model"; underestimating RAM |
| 5 | **Retrieval** | hybrid BM25 + dense + RRF; ACL filter **at retrieval** | `recall@k` | dense-only; filter after retrieve |
| 6 | **Reranking** | cross-encoder on the shortlist; latency cost | `nDCG@k` | rerank fixes broken recall (it can't) |
| 7 | **Generation** | ground-and-cite; abstention; verify citations | faithfulness | "use a better model" for hallucination |
| 8 | **Eval gate** | split retrieval vs generation; production-derived golden set; LLM-judge κ | all above | one end-to-end score; unvalidated judge |
| 9 | **Serving** | latency budget per stage; caching; tiered routing | p99, $/query | ignoring output-token cost |
| 10 | **Security** | retrieval-time ACLs; PII (3 stages); injection; audit | leak tests | ACLs in the prompt |
| 11 | **Ops / freshness** | CDC re-index; top-k-overlap telemetry; version pinning | drift | "add more replicas" for staleness |

---

## Numbers to say out loud

- **Index RAM (HNSW):** `N × dim × 4B × ~1.8`. 50M × 1,536-d ≈ **450–600 GB** → consider IVFPQ (~20× less ≈ 25–35 GB + reranker rescue).
- **Query latency:** ANN ~10ms + rerank ~120–150ms + generation → fits a 1.5s chat SLA.
- **Ingest (one-time):** `corpus_tokens × embed price` + contextualization (~$1.02/M doc tokens, ~90% off with prompt caching).
- **Re-embed (per TB):** meaningful monthly cost → **CDC incremental + dual index**, not nightly full rebuilds.

The point is to *reason* in these, not memorize them.

---

## Name the silent failures unprompted (the strongest signal)

Anyone can draw `embed → retrieve → generate`. The senior engineer names how it fails at 3am:

1. **Silent freshness drift** — stale/vendor-changed embeddings sag recall with green dashboards → top-k-overlap telemetry.
2. **Permission leak** — a missing ACL filter or injection pulling privileged context → filter-then-retrieve + validate retrieved content.
3. **Representation shearing** — an in-place re-embed mixes incomparable vector generations → dual index + atomic cutover.
4. **The long tail** — ~5% of queries drive ~80% of retrievals; failures hide in the rare 95% → canary on real traffic, error-analyze the tail.
5. **Cost cliff** — a reasoning model or uncapped reranker multiplies the bill → route the easy tail to a small model, cap candidates, cache hotspots.

---

## Reference code — ACL-tagged ingest + permission-aware answer

```python
# OFFLINE: idempotent, ACL-tagged, dual-index-safe ingest.
def ingest(doc, source_acl, llm, embed, index):
    text = parse_layout_aware(doc)                       # parsing is stage zero
    chunks = structure_aware_split(text, target=512)
    contextualized = contextualize(text, chunks, llm)    # orphaned-chunk fix, ~90% off w/ cache
    records = [{
        "text": c, "doc_id": doc.id, "chunk": i,
        "acl": source_acl,            # permission travels WITH the chunk
        "last_verified": now(),       # freshness you can time-filter on
    } for i, c in enumerate(contextualized)]
    index.upsert(embed([r["text"] for r in records]), records)   # embeddings + BM25

# ONLINE: filter at search time, never after.
def answer(query, user, llm):
    candidates = hybrid_search(query, k=50, acl_filter=user.allowed_acls)  # BM25 + dense + RRF
    context = rerank(query, candidates, top_n=5)                          # precision stage
    prompt = ("Answer ONLY from the numbered context. Cite the chunk id [n] after each claim. "
              "If the context does not contain the answer, reply: I don't know.\n\n"
              + format_with_ids(context) + f"\n\nQ: {query}")
    text = llm(prompt)
    valid = {c["id"] for c in context}
    cited = extract_citations(text)
    if not cited or not set(cited) <= valid:
        log_metric("ungrounded_or_bad_citation")         # gate / retry / flag
    return text, cited
```

---

## Worked designs against this rubric

- [QA over 10M documents](worked-qa-over-10m-docs.md)
- [Enterprise multi-tenant RAG](worked-enterprise-multi-tenant-rag.md)
- [Agentic RAG over tools](worked-agentic-rag-over-tools.md)
- [Private RAG for a regulated (HIPAA) customer](worked-private-rag-regulated.md)
