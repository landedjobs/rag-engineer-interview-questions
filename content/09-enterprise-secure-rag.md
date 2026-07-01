# Enterprise & Secure RAG

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md) · drill the [security question bank](../questions/security.md)*

Secure and enterprise RAG went from nice-to-have to a whole interview round in 2026 — especially for Forward-Deployed Engineer and enterprise AI roles. The deliverable is not "a vector DB and an LLM on the customer's cloud." It's **a control plane (identity, policy, audit, guardrails) wrapped around a data plane (ingest, retrieve, generate), with the model as a swappable config choice.** Every layer emits a signed, user-keyed artifact.

---

## Authorization at retrieval time — not in the prompt

The single most important rule: **compute the authorization decision in deterministic gateway code at retrieval time, which the model cannot bypass.** If a chunk is never retrieved, it can never leak. Two patterns:

| Pattern | How | Best when |
|---|---|---|
| **Pre-filter** | resolve the user's allowed doc-ID set first, pass it as an `$in` filter to the vector search | user sees a *small slice* of a large corpus |
| **Post-filter** | retrieve top-k, then re-check each hit against the authz graph | user sees *most* of the corpus |

```python
# Post-filter ReBAC: retrieve first, then re-check every hit against the authz graph.
def retrieve_authorized(query, user_id, k=20):
    hits = vector_search(query, k=k)          # vector DB knows nothing of identity
    allowed = []
    for h in hits:
        ok = authz.check(                     # microsecond-latency relationship check
            resource=f"document:{h['doc_id']}",
            permission="view",
            subject=f"user:{user_id}",
        )
        if ok:
            allowed.append(h)                 # only permitted chunks reach the prompt
    return allowed
# Pre-filter variant:
#   ids = authz.lookup_resources("document", "view", f"user:{user_id}")
#   hits = vector_search(query, k=k, filter={"doc_id": {"$in": ids}})
# Revoke access later? One relationship write -- the model never sees the doc again.
```

**Why ReBAC (computed-on-read) beats baked-in metadata ACLs:** a metadata ACL goes stale the instant a group membership changes — a revoked user keeps getting hits until you re-embed. A computed-on-read decision reflects the revocation *immediately* (one relationship write) at microsecond latency. Frameworks — SpiceDB/AuthZed, OpenFGA (CNCF), OPA — are interchangeable for this; **pick whichever the customer already operates**, don't introduce a second policy engine just for RAG. The pattern (per-chunk check, computed on read, in deterministic code) is identical.

> [!WARNING]
> **Pre-baked role tags are a pre-filter at best, never the authoritative gate.** OWASP LLM08 warns "embeddings strip context and classification" — the vector store is oblivious to labels unless the retriever re-checks. Perimeter tools (firewall, SIEM, DLP) *cannot see* retrieval-layer leakage: the request is an authenticated user on an approved endpoint, and DLP can't pattern-match natural-language privileged advice. Only a retrieval-path authorization check catches it. **The retriever is the ungoverned security boundary.**

---

## Selectivity and the filtered-ANN recall cliff

Pre vs post is a **selectivity decision** with a trap. If a user can see only ~0.5% of a 50M-doc corpus, a naive metadata filter that excludes 99.5% of candidates **wrecks HNSW recall** — the graph walk finds almost all neighbors filtered out and returns far fewer than k real hits. Fixes: **filter-aware / tenant-partitioned indexes**, or pre-resolve the allowed set and search only within it. Getting this wrong is how you crater recall while thinking you've only added a filter.

---

## Multi-tenant isolation — Silo / Pool / Bridge

The AWS-framed spectrum, and a favorite probe:

| Model | Isolation | Cost | Watch for |
|---|---|---|---|
| **Silo** | fully separate stack per tenant | highest | over-provisioning for low-tier tenants |
| **Pool** | shared end-to-end pipeline | lowest | no vector-store performance isolation; noisy-neighbor degrades everyone |
| **Bridge** | in between (~100 tenants) | medium | no per-tenant end-to-end encryption |

> [!TIP]
> "Always Silo" and "always Pool" are both weak answers. The senior response: **"it depends on tenant tier and compliance"** — Silo for regulated/high-tier, Pool for low-tier, Bridge between — and *compare* them on the cost vs isolation vs blast-radius axis. The 2026 pattern for the data layer is **pgvector + row-level security (RLS) + HNSW**; app-layer filtering alone is explicitly insufficient.

---

## SSO must reach the agent, not just the human

Enterprise auth is SAML/OIDC SSO bound to the customer's IdP (Okta, Entra ID) with just-in-time provisioning — and it must extend *down to the agent and its connectors*, not stop at the human. If the agent acts under a shared service account, every log line reads "service-bot did X" and you've lost the chain back to the human. With identity propagated end to end, the audit log says "user:kim, via the assistant, retrieved document:123." Scope the agent's token to exactly the tools and data the current user may touch (least privilege) so an over-broad service account never becomes an over-privileged god-account.

---

## PII redaction — three mandatory stages

Missing any one stage is a leak; each stage catches the prior stage's miss.

| Stage | Tooling | Failure if skipped |
|---|---|---|
| **1. Ingest (pre-index)** | Presidio, spaCy NER, per-class regex | PII embedded *into* vectors — leaks on any close semantic query, forever |
| **2. Retrieval (re-check)** | fast NER re-scan of retrieved chunks | stale/missing classification metadata still gets returned |
| **3. Output (gateway)** | vault-backed tokenization | sensitive content survives the prompt to the user |

> [!WARNING]
> **Embedding inversion** is the non-obvious fourth risk and a favorite advanced probe. ACL 2024–2025 research shows decoder-style attacks can reconstruct **50–70% of the original tokens from stolen vectors** (the ALGEN attack trains a transferable black-box inverter from ~1,000 samples). Your vector store is *not* a safe place for raw PII "because it's just numbers." Mitigate with **differential-privacy noise at embedding time** (at a measured recall cost you recover with a reranker) plus **encryption at rest**. When asked "if our vector DB is breached, what's the worst case?", name **inversion *and* poisoning** — not "the embeddings are meaningless."

There's a real tension: aggressive ingest redaction improves safety but **eats recall** — redact the counterparty name and a query needing exactly that entity retrieves nothing. Resolution: **per-data-class** redaction (hard-mask SSNs/card numbers; preserve searchable entities behind a vault token that keeps the embedding meaningful), applied aggressively only where the audit posture demands.

---

## Indirect prompt injection — the zero-click vector

The threat that makes guardrails non-optional. **Indirect prompt injection** rides inside *retrieved content*, not the user's message. **EchoLeak (CVE-2025-32711**, CVSS 9.3, patched June 2025) is the enterprise standard: an attacker emails hidden instructions telling the agent to scan recent messages for sensitive keywords and append them to an attacker URL. The user never interacts with the prompt — **retrieval of the email is the trigger.**

The defense is layered, not a single filter:

- **Instruction hierarchy:** `system prompt > retrieved context > user input`; untrusted retrieved content can never steer a privilege-bearing tool call.
- **Ingest sanitization:** parse into fixed-size passages, normalize Unicode (strip hidden chars), strip injection payloads.
- **Delimit data from instructions:** wrap retrieved text in tags the model is told to treat as quoted, never commands (the cheapest structural defense).
- **Least-privilege tool scope:** retrieved content cannot reach a high-privilege action.

> [!WARNING]
> **"Contextual grounding stops injection" is wrong.** The AWS caveat, verbatim: grounding "helps mitigate prompt injection by limiting responses to the retrieved context but *does not eliminate all injection vectors*." Indirect injection lives *inside* the retrieved context. And stronger guardrails create false positives — internal QA prompts saying "ignore the previous instructions" get refused; ship an **allow-list** for known-good internal patterns and tune thresholds. Removing the guardrail is the wrong fix.

---

## The 7 RAG security risks (map each to a control)

| Risk | Layer | Control | OWASP |
|---|---|---|---|
| Malicious content ingestion | ingest | sanitize + classify | LLM01 |
| Over-permissioned retrieval | retrieval | ReBAC pre/post-filter | LLM06 |
| Exfiltration via responses | output | output guardrail + audit | LLM02 |
| Indirect prompt injection | retrieval/runtime | `system > context > user` + tool-scope limits | LLM01 |
| Vector-DB poisoning | ingest/store | provenance + signed ingest + drift monitor | LLM08 |
| Sensitive leakage via embeddings | store | DP noise + encrypt at rest | LLM08 |
| Supply-chain (3rd-party deps) | build | pin + review deps | — |

---

## Deployment posture & the audit schema

**Posture spectrum:** SaaS (fastest, SOC-2 ceiling) → **cloud-managed via PrivateLink with the gateway in the customer VPC** (the default for regulated commercial customers — HIPAA-ready, ~2–4 week deploy, customer owns keys + vector store) → air-gapped (highest ceiling, months of lead time, heaviest ops). **Air-gap closes egress/residency only — it does *not* fix authorization**; you still need retrieval-time ACLs, PII redaction, and audit inside the perimeter. An EU regulator may also distinguish "data in jurisdiction" from "control plane in jurisdiction" — a US-operated control plane can fail even with EU data residency.

**The audit log a regulator wants** is append-only, signed events with `user_id`, `trace_id`, returned `doc_ids` + `chunk_ids`, the per-doc authz decision, and model/index versions — **referencing chunk text by ID, not copying it** (copying raw chunks creates a second PII store to re-protect).

---

## Interview angles

- **"Where do you enforce per-user document permissions?"** → at retrieval time, in deterministic code (pre/post-filter ReBAC) — before rerank/prompt assembly.
- **"Why ReBAC over baked-in metadata ACLs?"** → computed-on-read reflects revocations immediately; metadata goes stale until re-embed.
- **"Pick a multi-tenant isolation model."** → depends on tier/compliance — Silo/Pool/Bridge on the cost/isolation/blast-radius axis; pgvector + RLS + HNSW.
- **"If our vector DB is breached but only has embeddings?"** → embedding inversion (50–70% reconstruction) + poisoning; DP noise + encryption at rest.
- **"Classify this attack: emailed doc exfiltrates data when retrieved."** → indirect prompt injection (EchoLeak/CVE-2025-32711); instruction hierarchy + tool-scope limits.
- **"Is Presidio at ingest enough for PII?"** → no — three stages (ingest, retrieval re-check, output tokenization) + DP noise on vectors.

---

## 📚 Resources

- 📄 [Contextual Retrieval → Access Control Secrets for RAG](https://www.lmdconsulting.com/blogs/unlocking-secure-ai-access-control-secrets-for-rag-systems) — LMD Consulting (May 2025) · RBAC + ABAC for retrieval.
- 💻 [microsoft/presidio](https://github.com/microsoft/presidio) — **MIT** · PII detect/redact/mask/anonymize. *Adaptable with attribution; detection is best-effort — layer it.*
- 📄 [Multi-Tenant RAG with pgvector + RLS](https://medium.com/@rockingmanas78/how-to-create-a-multi-tenant-rag-44aa0fefa383) · [Building Multi-Tenant RAG with PostgreSQL](https://www.tigerdata.com/blog/building-multi-tenant-rag-applications-with-postgresql-choosing-the-right-approach) — TigerData · the schema/row/DB-per-tenant decision tree.
- 📄 [Multi-Tenant RAG: RLS in pgvector with MCP](https://blog.techtush.in/multi-tenant-rag-row-level-security-in-pgvector-with-mcp) — techtush (May 2026) · the 2026 isolation pattern.
- 📘 [OWASP LLM Top 10](https://genai.owasp.org/llm-top-10/) — the threat-model checklist (LLM01 injection, LLM02 disclosure, LLM06 excessive agency, LLM08 vector weaknesses).
- 📄 [EchoLeak (CVE-2025-32711)](https://arxiv.org/abs/2509.10540) · [Indirect Prompt Injection](https://www.lakera.ai/blog/indirect-prompt-injection) — Lakera · the zero-click indirect-injection standard.
- 📄 [Secure LLMs, RAG & Agentic AI](https://www.enkryptai.com/blog/enterprise-ai-security-framework-2025-securing-llms-rag-and-agentic-ai) — Enkrypt AI (Jun 2025) · OWASP-aligned enterprise framework.
- 📄 [Confidential RAG Pipeline (Intel TDX)](https://openmetal.io/resources/blog/how-to-build-a-confidential-rag-pipeline-that-guarantees-data-privacy/) — OpenMetal (Jan 2026) · air-gapped/confidential compute for regulated industries.
- 🎬 [DEF CON 33 — Exploiting Shadow Data from AI Models and Embeddings](https://youtube.com/watch?v=O7BI4jfEFwA) — the embedding/vector-store attack surface.
