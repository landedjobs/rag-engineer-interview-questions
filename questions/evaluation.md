# Evaluation — Questions

> *Part of the [RAG question bank](README.md) · pair with [content/05-rag-evaluation.md](../content/05-rag-evaluation.md)*

Legend: ✅ correct · ▫️ distractor · 🟢 Core · 🟡 Senior · 🔴 Staff · ✅ Reported / 🔮 Representative

---

## Multiple-choice checkpoints

**Q1 · 🟢 Answers are fluent but keep stating facts that aren't in the retrieved context. Which metric most directly flags this?**

- ▫️ Answer relevancy — *Measures whether the answer addresses the question, not whether its claims are supported.*
- ✅ Faithfulness / groundedness — *Checks that every claim is entailed by the retrieved context — though it degrades on numeric/multi-hop, so pair it with a confidence-aware check.*
- ▫️ Context recall — *Measures whether you fetched the needed context, not whether the answer stuck to it.*

**Q2 · 🔴 You build a golden set by having an LLM write Q&A from chunks you hand-picked. Offline recall is 0.91; production recall is 0.40. What went wrong?**

- ▫️ The production retriever is worse and needs a bigger embedding model — *The gap is a measurement artifact — you graded against contexts the live system never surfaces.*
- ✅ The gold contexts came from hand-picked chunks, not the production retriever, so offline eval measured a corpus the live system never uses — *Generate gold questions, but harvest gold contexts from real production retrieval runs.*
- ▫️ RAGAS computed faithfulness incorrectly — *This is a recall-provenance problem, not a metric bug.*

**Q3 · 🔴 A teammate uses GPT-4 as an LLM judge and reports its scores as the deploy gate, with no human comparison. Senior objection?**

- ✅ An unvalidated judge can be systematically wrong at scale — measure agreement with human ground truth (Cohen's κ) and calibrate before trusting it to gate — *The judge is itself a model output needing evaluation; raw % agreement flatters, so report κ and iterate the judge prompt.*
- ▫️ GPT-4 is too cheap to be a reliable judge — *Cost isn't the issue; uncalibrated biases (position, verbosity, self-preference) are.*
- ▫️ Judges should always use a 1–10 quality score for nuance — *Backwards — narrow binary judge questions are more reliable than vague 1–10 scores.*

**Q4 · 🟡 You're swapping embedding vendors; offline RAGAS scores improve. Safest way to ship?**

- ▫️ Ship it — offline scores went up — *Offline sets over-sample easy queries and can be flattered by the new vendor's own tuning; the long tail can still collapse.*
- ✅ Shadow/canary on real production traffic, watching per-span hit@k on the long tail before full rollout — *Embedding swaps are the silent-drift case; only the production distribution + per-span tracing reveals long-tail recall loss.*
- ▫️ Trust the vendor's benchmark numbers — *Vendor benchmarks rarely match your corpus or query distribution.*

**Q5 · 🔴 Your guardrail reliably blocks jailbreaks in the user prompt, but injecting documents into its context flips its judgments in a meaningful fraction of cases — and your retrieved chunks are exactly such documents. What's the lesson?**

- ▫️ The guardrail just needs a higher threshold — *Thresholds don't address the architectural gap: the retrieved-context channel isn't being checked at all.*
- ▫️ Disable retrieval for sensitive queries — *That guts the product; the fix is to validate the second input channel.*
- ✅ RAG has two input channels — the user prompt AND retrieved context; you must validate retrieved context (and whitelist tool calls), not just the user message — *Indirect prompt injection rides in through documents; guarding only the user side is negligent.*

---

## Reported & representative open questions

1. **Evaluate a RAG pipeline — which metrics (nDCG, MRR, precision@k, recall@k)?** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *recall@k as the ceiling; nDCG/MRR for ranking quality.*
2. **What is faithfulness in RAGAS vs context precision / recall?** — ✅ Reported ([Ragas docs](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/)) — *claim-entailment vs context relevance/coverage.*
3. **How do you calculate the accuracy of your agentic-research / RAG system?** — ✅ Reported ([Reddit r/LangChain — "grilled in an ML interview"](https://www.reddit.com/r/LangChain/comments/1k662xc/got_grilled_in_an_ml_interview_today_for_my/)) — *split retrieval vs generation, gate on a golden set.*
4. **Design offline vs online eval for production RAG.** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *CI gate on a frozen golden set; then shadow/canary on production traffic.*
5. **Design a hallucination-mitigation loop (faithfulness scoring, claim extraction, self-consistency).** — ✅ Reported ([Adil Shamim](https://adilshamim8.medium.com/every-ai-engineer-interview-question-you-need-to-know-in-2026-from-100-real-interviews-b5b7ae4b961a)) — *see [07-grounding](../content/07-grounding-citations-hallucination.md).*
6. **When do you pick Ragas, DeepEval, or TruLens?** — ✅ Reported ([DeepEval vs Ragas](https://deepeval.com/blog/deepeval-vs-ragas), [DeepEval vs TruLens](https://deepeval.com/blog/deepeval-vs-trulens)) — *Ragas (RAG metrics), DeepEval (15+ benchmarks, CI), TruLens (tracking + groundtruth).*
