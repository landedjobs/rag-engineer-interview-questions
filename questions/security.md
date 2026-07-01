# Security & Enterprise — Questions

> *Part of the [RAG question bank](README.md) · pair with [content/09-enterprise-secure-rag.md](../content/09-enterprise-secure-rag.md)*

Legend: ✅ correct · ▫️ distractor · 🟢 Core · 🟡 Senior · 🔴 Staff · ✅ Reported / 🔮 Representative

---

## Multiple-choice checkpoints

**Q1 · 🟢 Where should you enforce per-user document permissions in a RAG system?**

- ▫️ In the prompt — tell the model to ignore docs the user can't see — *Prompt-level rules leak: the chunks were still retrieved and can surface, and injection can override the instruction.*
- ✅ At retrieval time, as a metadata/ACL filter on the search (before rerank and prompt assembly) — *If a chunk is never retrieved, it can never leak.*
- ▫️ After generation, by redacting the answer — *Too late — the model already saw forbidden context and may have used it.*

**Q2 · 🟡 A multi-tenant RAG returns correct answers in testing. What's the highest-priority production risk to design against?**

- ▫️ Slightly higher latency from metadata filtering — *Real but minor next to a data-isolation breach.*
- ✅ Permission leaks — enforce ACL filters at retrieval time, before rerank/prompt assembly — *Cross-tenant leakage (filter bypass or injection pulling privileged context) is the catastrophic failure.*
- ▫️ Choosing the wrong embedding model — *Matters for quality, not the security/isolation risk that defines a multi-tenant system.*

**Q3 · 🔴 A junior analyst's RAG query returns a privileged memo they're not staffed on, yet firewalls, SIEM, and DLP fired no alert. What does this tell you, and what's the fix?**

- ▫️ DLP just needs better regex patterns to catch the memo — *The leaked content is natural-language privileged advice, not a structured PII signature — DLP fundamentally cannot see this class of leak.*
- ✅ Perimeter tools can't see retrieval-layer leakage; enforce identity on every retrieved document (pre/post-filter ReBAC) before chunks reach the prompt — *The retriever is the ungoverned security boundary.*
- ▫️ Restrict who can open the chat assistant — *The analyst is legitimately allowed to use it; the failure is document-level authorization at retrieval.*

**Q4 · 🔴 Your corpus has 50M docs and each user sees under 1%. Which authorization pattern fits, and what must you watch for?**

- ▫️ Post-filter: retrieve top-k, then drop the docs the user can't see — *With <1% visible, almost every hit is denied — wasteful and low-recall.*
- ✅ Pre-filter to the allowed ID set, using a tenant-partitioned/filter-aware index to avoid the ANN recall cliff — *High selectivity favors pre-filtering; a filter excluding ~99% craters naive HNSW recall, so partition the index.*
- ▫️ Bake the allowed roles into each chunk's metadata at ingest and rely on that alone — *Pre-baked ACLs go stale on every group change; a pre-filter at best, never the authoritative gate.*

**Q5 · 🔴 A customer asks: "If our vector database is breached but contains only embeddings, what's the real exposure?"**

- ▫️ Embeddings are just numbers, so a breach reveals nothing useful — *False safety — decoder-style inversion reconstructs 50–70% of original tokens, and poisoning is a separate risk.*
- ✅ Embedding inversion can reconstruct much of the original text (50–70%), and the store is a poisoning target — mitigate with DP noise at embedding time and encryption at rest — *Names the real exposure (inversion + poisoning) and the validated mitigations.*
- ▫️ The only risk is that the attacker learns how many documents you have — *Drastically understates it.*

**Q6 · 🔴 An attacker emails a document containing hidden instructions; when the agent later retrieves it, it exfiltrates data to a URL — the user never typed anything malicious. What class of attack, and the core defense?**

- ▫️ Direct prompt injection; fix by sanitizing the user's input field — *The user's input was benign; the payload arrived via retrieved content.*
- ✅ Indirect prompt injection (CVE-2025-32711 class); treat retrieved context as untrusted (system > context > user), strip payloads at ingest, constrain tool scope — *The instruction hierarchy + least-privilege tool scope stop untrusted context from steering a privileged call.*
- ▫️ A model hallucination; fix with a larger model — *This is an adversarial injection, not a hallucination.*

**Q7 · 🟡 During a pilot you add Presidio redaction at ingest and call PII handling done. Senior critique?**

- ✅ Ingest redaction is best-effort and embeddings strip classification — you also need a retrieval re-check and output tokenization, plus DP noise/encryption on the vectors — *Single-stage redaction leaks on the miss; defense in depth across three stages is required.*
- ▫️ Presidio is sufficient because it's a Microsoft-supported framework — *Tool pedigree is irrelevant; Microsoft itself says detection is best-effort.*
- ▫️ Redaction is unnecessary if the deployment is in a VPC — *VPC isolation addresses egress, not PII inside the corpus.*

**Q8 · 🟡 An interviewer asks you to choose a multi-tenant isolation model for a RAG product with mixed customer tiers. Strongest response?**

- ▫️ Always use Pool — sharing the whole pipeline is cheapest — *Pool gives no vector-store performance isolation and lets a noisy neighbor degrade everyone.*
- ▫️ Always use Silo — full isolation is the only safe option — *Silo is the most expensive and rarely justified for low-tier tenants.*
- ✅ It depends on tenant tier and compliance: Silo for regulated/high-tier, Pool for low-tier, Bridge between — trading cost vs isolation vs blast radius — *Naming Silo/Pool/Bridge and comparing them is the senior signal.*

**Q9 · 🟡 A CISO objects: "Why can't we just use ChatGPT Enterprise for this regulated corpus?" Best response?**

- ✅ Name the three missing controls — no row-level tenant control over the corpus, no audit trail of input content, no PII redaction layer — plus the data-residency tests it fails — *Specific missing controls for THIS corpus, not a blanket ban; a US-operated control plane can also fail strict residency.*
- ▫️ ChatGPT Enterprise is simply not secure and should never be used — *Too blunt and wrong — it's SOC 2 and SSO-bound and fine for many use cases.*
- ▫️ A bigger or newer model would solve their concerns — *Model capability is unrelated to tenant isolation, audit, residency, and redaction.*

**Q10 · 🟡 Why is computing the authorization decision on read (ReBAC) preferable to precomputing ACLs into vector metadata?**

- ▫️ It's faster, because metadata filters are always slower than graph checks — *Speed isn't the point — metadata filters can be very fast. The issue is correctness over time.*
- ▫️ It avoids embeddings entirely, removing the vector DB from the auth path — *ReBAC runs alongside retrieval; embeddings still do the semantic matching.*
- ✅ A computed-on-read decision reflects revocations immediately (one relationship write), while baked metadata goes stale until a full re-embed — *Group membership churns; a precomputed gate is wrong the moment access changes.*

**Q11 · 🔴 You're designing the audit log. Which design best satisfies a SOC 2 / EU AI Act auditor without creating a second PII store?**

- ✅ Append-only, signed events with user_id, trace_id, returned doc_ids + chunk_ids, the per-doc authz decision, and model/index versions — referencing chunk text by ID, not copying it — *Replayable, tamper-evident, ties every retrieval to an identity and a policy decision, and avoids duplicating sensitive chunk text.*
- ▫️ Log the full retrieved chunk text for every request so nothing is lost — *Copies the sensitive corpus into your logs — a new leakage surface, not a control.*
- ▫️ Log only aggregate daily counts of queries and answers — *Aggregates can't reconstruct which documents a specific user retrieved.*

**Q12 · 🔴 A regulated customer says "your design still sends our queries over the public internet to the model." Using AWS, the precise answer?**

- ✅ A PrivateLink interface endpoint to Bedrock keeps traffic on the AWS backbone with no internet gateway or public IP, and customer KMS holds the keys — *PrivateLink resolves it exactly; "managed model" is not "public internet."*
- ▫️ Encrypt the traffic with TLS so it's safe in transit over the internet — *TLS protects confidentiality in transit but the traffic still traverses the public internet.*
- ▫️ Move to a fully air-gapped deployment immediately — *A months-long answer to an objection PrivateLink already resolves.*

**Q13 · 🟡 A customer insists on a fully air-gapped deployment "to be maximally secure" — no ITAR data, no strict residency, launch in a month. Senior response?**

- ▫️ Agree and start the air-gap build immediately — *Months of lead time and full SRE ownership; with no trigger and a one-month timeline it optimizes fear.*
- ✅ Probe for a real trigger; absent one, recommend SaaS-managed model via PrivateLink with the gateway in their VPC, and note air-gap doesn't fix authorization anyway — *Air-gap is justified only by specific triggers; document-level ACLs are still needed regardless.*
- ▫️ Tell them air-gap is unnecessary because the cloud is always secure — *Dismissive and wrong — air-gap is right for genuine triggers.*

**Q14 · 🟡 After enabling strict injection guardrails, your own internal QA prompts that say "ignore the previous instructions" start getting refused. Right fix?**

- ✅ Ship an explicit allow-list for known-good internal patterns and tune thresholds, accepting the quality/false-positive tradeoff — *Strong guardrails over-refuse; an allow-list + threshold tuning manages it without dropping protection.*
- ▫️ Remove the injection guardrail since it produces false positives — *Reopens the real attack surface; the issue is calibration.*
- ▫️ Tell QA to never use that phrase again — *A band-aid that recurs with other legitimate phrasings.*

---

## Reported & representative open questions

1. **Protect sensitive data in a RAG pipeline (PII redaction, RBAC, encryption).** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a); [Enkrypt AI](https://www.enkryptai.com/blog/enterprise-ai-security-framework-2025-securing-llms-rag-and-agentic-ai)) — *retrieval-time ACLs + three-stage PII + DP noise/encryption.*
2. **Design a multi-tenant RAG with full per-tenant isolation.** — ✅ Reported ([pgvector RLS](https://blog.techtush.in/multi-tenant-rag-row-level-security-in-pgvector-with-mcp); [TigerData](https://www.tigerdata.com/blog/building-multi-tenant-rag-applications-with-postgresql-choosing-the-right-approach)) — *pgvector + RLS + HNSW; Silo/Pool/Bridge; see the [worked design](../answers/worked-enterprise-multi-tenant-rag.md).*
3. **How would you handle a RAG query touching sensitive/PII data mid-conversation?** — ✅ Reported ([Reddit r/LangChain thread](https://www.reddit.com/r/LangChain/comments/1k662xc/got_grilled_in_an_ml_interview_today_for_my/)) — *retrieval re-check + output tokenization; per-data-class redaction.*
4. **Is contextual grounding enough to prevent prompt injection?** — 🔮 Representative — *no — it limits responses to retrieved context but indirect injection rides inside it; layer guardrails + tool-scope limits.*
5. **What does a regulator-grade audit log for RAG contain?** — 🔮 Representative — *append-only signed events: user_id, trace_id, doc_ids + chunk_ids, per-doc authz decision, model/index versions — referencing chunk text by ID, not copying it.*
