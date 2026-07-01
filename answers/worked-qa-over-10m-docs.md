# Worked Design — QA over 10M Documents

> *Scored against [the RAG system-design rubric](rag-system-design-rubric.md) · [content index](../README.md)*

**Prompt:** "Design a Q&A assistant over 10M articles. Users ask factual questions; it must be accurate or admit ignorance. 1.5s p95 budget."

---

## Step 0 — Clarify (collapse the branches)

- **Shape?** Chat over a large closed corpus — not web-scale. → dominant levers are indexing quality + reranking, latency budget is comfortable.
- **Scale?** ~10M docs → ~50M chunks at 256–512 tokens; assume moderate QPS with a hot tail.
- **Freshness?** Assume docs change daily with some contradictions → CDC re-indexing, `last_verified`.
- **Tenancy?** Single org, no per-user permissions here (see the [multi-tenant design](worked-enterprise-multi-tenant-rag.md) for that).
- **Stakes?** "Accurate or admit ignorance" → strict grounding + abstention + a faithfulness gate.

---

## Three swim-lanes

**Offline ingest (quality):** layout-aware parse → structure-aware chunk (~512, drop toward 256 if `precision@5` sags) → **contextual retrieval** (prepend LLM-written context, ~90% off with prompt caching) → embed (domain-fit model from MTEB) + index into HNSW **and** BM25 → attach `{doc_id, chunk, last_verified}`.

**Online query (latency):** hybrid retrieve top-50 (BM25 + dense → RRF) → cross-encoder rerank to top-5 → generate grounded, cite per claim, verify citations exist. Semantic cache in front for the hot tail.

**Deploy gate (safety):** split-eval on a production-derived golden set (retrieval `recall@k` + generation faithfulness), p99 latency check, pinned model/index versions.

---

## Index economics — the arithmetic they'll push on

```
50M chunks × 1,536-d × 4 bytes ≈ 300 GB raw
× ~1.8 for the HNSW graph        ≈ 450–600 GB
```

That's a large RAM bill. At this scale, **weigh IVFPQ + a reranker**: ~20× less memory (~25–35 GB) at a recall cost the cross-encoder shortlist recovers. Decision: if the corpus fits a single managed HNSW instance in budget, keep HNSW for simplicity; otherwise IVFPQ + rerank, or **shard** the index (by topic/hash) for both scale and parallel recall.

| Lever | Effect |
|---|---|
| **Sharding** | parallelize recall; fit RAM per node; scale horizontally |
| **IVFPQ + reranker** | ~20× RAM cut; recover precision on the shortlist |
| **Semantic cache** | absorb the hot tail (~5% of queries → ~80% of retrievals) |
| **CDC re-index** | keep 10M docs fresh incrementally, not nightly rebuilds |

---

## Latency budget (1.5s p95)

| Stage | Budget |
|---|---|
| embed query | ~15ms |
| hybrid retrieve (ANN + BM25) | ~40–80ms |
| rerank (50 candidates) | ~120ms |
| generate (streamed, capped output) | remainder — stream so TTFT is low |

Trim **output tokens** (they cost ~4× input and dominate latency), stream, and cache the hot tail.

---

## Accurate-or-admit-ignorance

Grounding contract: answer only from the numbered context, cite the chunk id per claim, reply "I don't know" when the context doesn't support an answer, and **verify cited ids exist**. Wire **faithfulness as an online guardrail** — block or route-to-human below the SLO. For daily-changing docs with contradictions, prefer the chunk with the freshest `last_verified` and surface the conflict rather than silently picking one.

---

## Silent failures (name them unprompted)

- **Freshness drift** — top-k-overlap telemetry on a fixed probe set; CDC re-embedding.
- **Representation shearing** — never re-embed in place; dual index + atomic cutover.
- **The long tail** — canary on real traffic; error-analyze the rare 95%.
- **Cost cliff** — cap reranker candidates; route easy queries to a small model; cache hotspots.

---

## Rubric scorecard

| Stage | Decision |
|---|---|
| Framing | closed chat shape, single-tenant, accurate-or-abstain |
| Chunking | structure-aware + contextual, sized by `precision@5` |
| Index | HNSW if it fits; else IVFPQ + rerank / sharding |
| Retrieval | hybrid BM25 + dense + RRF, top-50 |
| Rerank | cross-encoder to top-5 (~120ms) |
| Generation | ground-and-cite + abstention + citation verification |
| Eval | split eval on production-derived golden set + faithfulness gate |
| Serving | per-stage budget, semantic cache, streamed output |
| Ops | CDC re-index, freshness telemetry, version pinning |
