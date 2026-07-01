# Worked Design — Enterprise Multi-Tenant RAG

> *Scored against [the RAG system-design rubric](rag-system-design-rubric.md) · pair with [09-enterprise-secure-rag](../content/09-enterprise-secure-rag.md)*

**Prompt:** "Design a B2B RAG product. Multiple customers, ~2–5M docs each, per-user document permissions, exact part-number queries common, 1.5s budget, strict isolation, weekly doc updates."

---

## Step 0 — Clarify

- **Shape?** Enterprise search over permissioned corpora.
- **Scale?** Per-tenant ~2–5M docs; each user sees a *small slice* → **high selectivity** → pre-filter.
- **Freshness?** Weekly updates → CDC re-index, not full rebuilds.
- **Tenancy?** Multi-tenant with per-user ACLs → the defining constraint. Ask about compliance tier (drives Silo/Pool/Bridge).
- **Stakes?** A cross-tenant leak is catastrophic → retrieval-time ACLs are non-negotiable.

---

## The isolation decision (Silo / Pool / Bridge)

| Model | Use for | Cost |
|---|---|---|
| **Silo** — separate stack/index per tenant | regulated / high-tier tenants | highest |
| **Pool** — shared pipeline | low-tier tenants | lowest; no vector-store perf isolation |
| **Bridge** — in between (~100 tenants) | mid-tier | medium; no per-tenant E2E encryption |

**Decision:** tier-dependent. Silo (per-tenant namespace/index) for regulated customers — no cross-tenant vector can ever be returned. Pool for low-tier. The data layer for a single tenant: **pgvector + row-level security (RLS) + HNSW** — app-layer filtering alone is explicitly insufficient.

---

## Authorization — at retrieval time, in deterministic code

Each user sees <1% of their tenant's corpus → **pre-filter**: resolve the allowed doc-ID set (ReBAC via SpiceDB/OpenFGA/OPA — whichever the customer already runs), pass it as an `$in` filter to the vector search, then **post-filter** re-check the hits.

> **The filtered-ANN recall cliff:** a metadata filter that excludes ~99% of candidates craters naive HNSW recall (the graph walk finds almost everything filtered out). Fix: **partition the index by tenant/department** (filter-aware index) or pre-resolve the allowed set and search only within it.

```python
def retrieve_authorized(query, user, k=50):
    allowed_ids = authz.lookup_resources("document", "view", f"user:{user.id}")  # pre-filter
    hits = tenant_index(user.tenant).hybrid_search(
        query, k=k, filter={"doc_id": {"$in": allowed_ids}})   # partitioned, filter-aware
    return [h for h in hits if authz.check(h["doc_id"], "view", user.id)]  # post-filter re-check
```

Why ReBAC over baked-in metadata ACLs: **computed-on-read reflects revocations immediately** (one relationship write); baked metadata goes stale until a re-embed.

---

## Exact part-number queries → hybrid is mandatory

Part numbers like "ACME-X200" are dense retrieval's weak spot. **Hybrid BM25 + dense + RRF** — and check the BM25 analyzer doesn't strip the hyphen (the classic gotcha that makes exact-token queries silently miss).

---

## Pipeline

**Ingest:** parse → structure-aware + contextual chunk → embed → index per tenant, with `acl` metadata on every chunk. **CDC weekly** re-embed of changed docs behind a dual index.

**Query:** SSO/JWT resolves identity → pre-filter to allowed set → hybrid retrieve top-50 → post-filter re-check → rerank to 5 → grounded cited answer.

**Gate:** split eval per tenant + a **quarterly permission-leak red-team** (try to retrieve a forbidden doc phrased 20 ways — anything that returns a forbidden chunk is a re-authorization bug).

---

## Audit

Append-only, signed events: `user_id`, `trace_id`, returned `doc_ids` + `chunk_ids`, per-doc authz decision, model/index versions — **referencing chunk text by ID, not copying it** (copying raw chunks creates a second PII store).

---

## Silent failures

- **Permission leak** via stale metadata or injection pulling privileged context → post-filter re-check + validate retrieved content + quarterly red-team.
- **Recall cliff** from high-selectivity filters → partitioned/filter-aware index.
- **Freshness drift** on weekly updates → top-k-overlap telemetry + CDC.

---

## Rubric scorecard

| Stage | Decision |
|---|---|
| Framing | multi-tenant enterprise search, per-user ACLs, exact-token queries |
| Isolation | Silo per regulated tenant / Pool low-tier; pgvector + RLS + HNSW |
| Authorization | pre-filter to allowed IDs + post-filter ReBAC re-check, at retrieval time |
| Retrieval | hybrid BM25 + dense + RRF (analyzer-safe for part numbers), partitioned index |
| Rerank | cross-encoder to 5 |
| Generation | ground-and-cite over permitted chunks only |
| Security | retrieval-time ACLs, signed audit-by-ID, quarterly leak red-team |
| Ops | CDC weekly re-index, dual index, freshness telemetry |
