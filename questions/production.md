# Production, Latency & Cost — Questions

> *Part of the [RAG question bank](README.md) · pair with [content/08-production-latency-cost.md](../content/08-production-latency-cost.md)*

Legend: ✅ correct · ▫️ distractor · 🟢 Core · 🟡 Senior · 🔴 Staff · ✅ Reported / 🔮 Representative

---

## Multiple-choice checkpoints

**Q1 · 🔴 Latency, throughput, and individual retrievals all look healthy, but users say answers are subtly out of date. Dashboards are green. Likely failure and how you'd catch it?**

- ▫️ A capacity problem — add more replicas — *Throughput is healthy; adding replicas does nothing for stale or drifted representations.*
- ✅ Silent freshness drift (stale docs or a vendor checkpoint change); catch it with top-k-overlap telemetry on a fixed probe set — *Recall can decay distributionally while per-request metrics stay green; only freshness/overlap telemetry reveals it.*
- ▫️ The reranker is misconfigured — *A reranker issue shows as precision problems on current docs, not systematically outdated answers.*

**Q2 · 🟡 A math/multi-step feature is slow and expensive. A teammate sends 2,000-token prompts but caps answers at 100 tokens; another sends 1,000-in / 1,000-out. Which is cheaper/faster and why?**

- ▫️ The first — shorter prompts are always cheaper — *Prompt length isn't the dominant cost; output tokens are pricier (~4×) and dominate latency.*
- ✅ The 2,000-in/100-out call — output tokens cost ~4× input and dominate total latency, so a short output beats a short prompt — *Output is the expensive, sequential part; and the long prompt may be cacheable.*
- ▫️ They cost the same — only total tokens matter — *Input and output are priced differently (output ~3–5× input), so the split matters.*

**Q3 · 🟡 A user reports the assistant "got noticeably worse this week," but you shipped nothing. Most likely cause and safeguard?**

- ▫️ Random bad luck — LLMs vary, nothing to do — *A sustained quality drop with no deploy is the classic signature of a provider-side model change or a stale index.*
- ✅ The provider silently updated the model (or the index went stale); pin the model version and run an eval gate + freshness telemetry — *Pinned versions + a golden-set eval + top-k-overlap catch silent regressions before users do.*
- ▫️ Your context window shrank — *Context limits don't silently shrink; an unannounced update is the usual culprit.*

**Q4 · 🟡 Your semantic cache is serving a wrong answer to a subtly different question. What's the fix?**

- ✅ Raise the similarity threshold and invalidate on index updates — never cache across tenants — *Too loose a threshold serves stale/wrong answers; tighten it, invalidate on updates, and respect isolation boundaries.*
- ▫️ Disable the cache entirely — *That throws away real cost/latency wins; the issue is calibration, not the cache.*
- ▫️ Cache only exact string matches — *That collapses hit-rate to near zero; semantic caching's value is the near-duplicate tail — tune the threshold instead.*

**Q5 · 🔴 A customer reports the assistant "gives wrong answers sometimes," but you can't read their confidential documents or outputs on-site. What lets you diagnose fastest?**

- ▫️ Ask the customer to paste failing answers and documents into a ticket — *Often impossible on a regulated site, and it doesn't scale.*
- ✅ Per-request traces — retrieval set, relevance scores, prompt version, model version, grounding score — so you localize retrieval vs prompt vs model without seeing the content — *The trace reveals "retrieval was fine, the prompt lacked a directive" without exposing the confidential payload.*
- ▫️ Increase the model size to reduce wrong answers — *Blind escalation without localizing the failure.*

**Q6 · 🟡 You want low-faithfulness answers to never reach users. Right use of your faithfulness metric?**

- ✅ Wire it as a guardrail: block or route-to-human any response scoring below the SLO, not just chart it — *An evaluator in the request path is what stops bad answers; a dashboard nobody acts on blocks nothing.*
- ▫️ Display it on a daily dashboard for the team to review — *Catches trends after the fact but lets every low-faithfulness answer ship in the meantime.*
- ▫️ Only compute it offline during pre-deployment evals — *Offline-only misses production drift and the specific bad answers users hit live.*

---

**Q7 · 🟡 An interviewer says "it works in the demo — how do you know it's production-ready?" Best answer?**

- ▫️ The answers are fluent and the stakeholders are happy — *That's the demo; fluency isn't faithfulness, and a few happy queries don't cover the long tail.*
- ✅ It passes a split-eval gate on a production-derived golden set, clears injection tests, and meets the p99 latency/cost budget — *Production-ready is a measurable bar across retrieval, generation, safety, and performance.*
- ▫️ It uses the latest models and a managed vector DB — *Tooling choices don't prove correctness, safety, or regression-resistance.*

**Q8 · 🔴 Asked to size the index for 50M chunks of 1,536-d embeddings on HNSW, what do you say?**

- ▫️ "A few gigabytes — embeddings are small" — *Off by two orders of magnitude; underestimating RAM forces a mid-project re-architecture.*
- ▫️ "It depends on the model" — *True but evasive — the interviewer wants the arithmetic, which you can do from dimension and count.*
- ✅ "~50M × 1,536 × 4 bytes ≈ 300 GB raw, ~450–600 GB with the HNSW graph — at this scale I'd weigh IVFPQ + a reranker" — *Doing N × dim × 4B × ~1.8 on the spot and connecting it to the index choice is the senior signal.*

## Reported & representative open questions

1. **Optimize RAG latency in production (caching, pre-filtering, model choice).** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *attribute a per-stage budget; trim output tokens; stream; pre-filter; cache; tiered routing.*
2. **What is semantic caching; how does it reduce cost/latency?** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a); [GPTCache](https://github.com/zilliztech/gptcache)) — *cosine-threshold cache over query embeddings; invalidate on updates.*
3. **Design a RAG that maintains context across multi-turn conversations.** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *query rewriting to resolve references; session memory; retrieve on the rewritten query.*
4. **Design a 50k-article support assistant: accurate-or-admit-ignorance, KB changes daily with contradictions.** — ✅ Reported ([KORE1 AI Engineer 2026](https://www.kore1.com/ai-engineer-interview-questions-2026/)) — *grounding + "I don't know" path; CDC re-index; contradiction handling; see the [rubric](../answers/rag-system-design-rubric.md).*
5. **Implement an `/ask` endpoint with a tiny agentic RAG in 30–60 minutes.** — ✅ Reported ([Reddit r/LLMDevs — interview prep](https://www.reddit.com/r/LLMDevs/comments/1piahou/interview_prep/)) — *retrieve → ground → cite, with a stop condition.*
6. **How do you know a RAG system is production-ready, not just demo-ready?** — 🔮 Representative — *split-eval gate on a production-derived golden set + injection tests + p99 budget.*
