# Worked Design — Private RAG for a Regulated (HIPAA) Customer

> *Scored against [the RAG system-design rubric](rag-system-design-rubric.md) · pair with [09-enterprise-secure-rag](../content/09-enterprise-secure-rag.md)*

**Prompt:** "A healthcare customer wants your RAG assistant deployed into their environment. PHI in the corpus, HIPAA readiness, ~50M chunks, users each see a small departmental slice, launch in ~3–4 weeks."

---

## Step 0 — Clarify (30 seconds in)

- **Users & PHI boundary?** Who queries, which docs contain PHI, where's the redaction line.
- **Compliance / residency?** HIPAA-ready; is there a strict data-residency or no-egress mandate? (drives deployment posture)
- **Per-source freshness SLA?**
- **Auth boundary?** SSO/IdP, per-department ACLs.
- **Affordable posture?** SRE burden they can carry.

> The deliverable is **not** "a vector DB + an LLM on their cloud." It's a **control plane** (identity, policy, audit, guardrails) wrapped around a **data plane** (ingest, retrieve, generate), with the model as a swappable config choice.

---

## Deployment posture

| Posture | Ceiling | Timeline | When |
|---|---|---|---|
| SaaS + SSO | SOC-2 | days | usually fails a HIPAA/PHI bar |
| **Cloud-managed via PrivateLink, gateway in customer VPC** | **HIPAA-ready** | **~2–4 weeks** | **the default here** |
| Air-gapped, open-weights | highest | 3–6+ months | only with a real trigger (ITAR, strict residency, no-egress) |

**Decision:** with a 3–4 week timeline and no stated air-gap trigger, recommend **cloud-managed LLM via PrivateLink with the gateway in their VPC** — customer owns keys (KMS) and the vector store; you carry the gateway. PrivateLink resolves "queries traverse the public internet" (no IGW/NAT/public IP). Note that **air-gap wouldn't fix authorization anyway** — you still need retrieval-time ACLs, redaction, and audit inside the perimeter.

---

## The request lifecycle (walk it end to end)

```
SSO/JWT  →  gateway resolves permissions  →  input guardrail
        →  retrieve + post-filter ReBAC   →  PII re-check
        →  generate from permitted chunks →  output guardrail + tokenize
        →  append signed audit event
```

Every layer produces its artifact; the failure of each control is the input to the next — the boundary is the *composition*, not any single control.

---

## PII — three mandatory stages (each catches the prior miss)

| Stage | Tooling | Catches |
|---|---|---|
| **1. Ingest** | Presidio + NER + per-class regex | PHI embedded into vectors (leaks forever on close queries) |
| **2. Retrieval re-check** | fast NER on retrieved chunks | classification lost between ingest and retrieval |
| **3. Output** | vault-backed tokenization | PHI surviving the prompt to the user |

Plus the non-obvious control: **differential-privacy noise at embedding time + encryption at rest** — because embedding inversion reconstructs 50–70% of original tokens from stolen vectors. Make redaction **per-data-class** (hard-mask SSNs/MRNs; keep searchable entities behind a vault token) to avoid gutting recall.

---

## Selectivity & isolation

Users see a small departmental slice of 50M chunks → **partition the index by department, pre-filter to the allowed set, post-filter with ReBAC** — avoiding the filtered-ANN recall cliff and enforcing isolation. Silo posture for this regulated tenant.

---

## Guardrails & injection

Three-layer defense in depth: input guardrail (schema + injection detection) → retrieval guardrail (ACL filter + similarity threshold) → output guardrail (groundedness + policy). Treat retrieved context as **untrusted** (`system > context > user`); strip payloads at ingest; constrain tool scope. Ship an **allow-list** for known-good internal QA prompts (strong guardrails over-refuse). Grounding helps but does **not** eliminate injection.

---

## Observability (self-hosted in the VPC)

Pick the spine first and **self-host it in the VPC** (e.g. Langfuse: ClickHouse + object storage + async queue). Capture every retrieval/LLM/tool step; export only **aggregated metrics** outward. Per-request traces (retrieval set, scores, prompt/model versions, grounding score) let you diagnose failures **without seeing confidential content** — the "retrieval was fine, the prompt lacked a directive" localization.

---

## A named security trade-off (what the rubric rewards)

> "We added differential-privacy noise to the PHI embeddings to defend against inversion of the corpus, accepting a measured recall hit we recovered with a reranker — chosen because a breach of raw vectors was the higher risk." — a named, defended trade-off with its cost mitigated. ("The design was secure by default" is the weak-signal answer.)

---

## Rubric scorecard

| Stage | Decision |
|---|---|
| Framing | control plane + data plane; PHI boundary, residency, posture, auth clarified |
| Posture | PrivateLink + gateway in customer VPC; customer owns keys + store |
| Isolation | Silo; partition by department; pre-filter + post-filter ReBAC |
| PII | three stages + DP noise + encryption at rest, per-data-class |
| Injection | 3-layer guardrails, instruction hierarchy, allow-list, tool-scope limits |
| Audit | append-only signed events by chunk ID, tied to identity + policy decision |
| Observability | self-hosted spine in VPC; aggregates-only egress |
| Trade-off | DP noise vs recall, defended and mitigated |
