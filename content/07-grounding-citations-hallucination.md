# Grounding, Citations & Hallucination

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md)*

The whole point of RAG is answers that are *accurate and attributable*. This topic is where behavioral rounds live ("debug a hallucination in production") and where the single most important reframe in RAG interviews applies: **most "hallucinations" are retrieval misses.**

---

## The reframe: hallucination is usually a retrieval failure

If the right chunk never made top-k, no generator can recover information that isn't in its context. Before touching the prompt or reaching for a bigger model, **localize**: is the gold chunk in the retrieved set or not?

- **Not retrieved** → a retrieval problem. Fix chunking / hybrid / ANN-params / reranker ([01](01-chunking.md)–[04](04-reranking-and-query-rewriting.md)).
- **Retrieved but the model ignored / contradicted it** → a *grounding* problem. Fix the grounding prompt or measure faithfulness.

> [!TIP]
> "The model hallucinated, so I'd use a bigger model" is the answer that ends the round. The senior reflex is to split the eval ([05](05-rag-evaluation.md)) and name the stage first. A bigger generator can't retrieve a chunk that was never fetched.

---

## Grounding: constrain the answer to the context

Grounding is instructing (and structuring) the model to answer *only* from retrieved context and to say "I don't know" when the context doesn't support an answer — the "accurate-or-admit-ignorance" behavior interviewers ask for by name.

Practical levers:

- **A strict grounding instruction:** "Answer only using the provided context. If the context does not contain the answer, say you don't know. Do not use outside knowledge."
- **Structured output with a rationale-before-answer field** so the model commits its evidence first (chain-of-thought in schema).
- **A faithfulness guardrail** (below) that blocks low-scoring answers rather than just charting them.

---

## Citations & attribution

Citations turn a search engine into a *trustable* answer engine. Two families:

| Approach | How | Tradeoff |
|---|---|---|
| **Generation-time** | model emits `[doc_id]` markers as it writes | tightly bound to the text, but the model can mis-cite |
| **Post-hoc** | after generation, match each claim back to a source chunk | verifiable and correctable, but an extra pass |

The 2026 research (REASONS benchmark, CiteFix) shows **post-hoc correction meaningfully improves citation accuracy** — worth knowing that generation-time citations are not automatically trustworthy and can be repaired after the fact.

> [!TIP]
> Cite at the **chunk** level (doc + chunk/span id), not the whole document — it's what makes an answer auditable and lets a user verify the claim. This is also why long-context "stuff everything" loses on attribution: there's no chunk to point at.

---

## Faithfulness as an *online* guardrail, not a dashboard

Faithfulness/groundedness scores every claim in the answer against the retrieved context. The senior move is to **wire it into the request path**: block or route-to-human any response scoring below an SLO — not just chart it on a dashboard nobody acts on. A dashboard catches trends *after* the fact; a guardrail stops the bad answer *before* the user sees it.

```python
# Faithfulness as a guardrail in the request path -- not a chart.
def answer(query, ctx, llm, judge, slo=0.85):
    resp = generate(query, ctx, llm)
    score = judge.faithfulness(resp, ctx)     # every claim entailed by ctx?
    if score < slo:
        return route_to_human(query, ctx, resp)   # block, don't ship a low-faithfulness answer
    return resp
```

Remember the blind spot from [05](05-rag-evaluation.md): faithfulness *nulls out* on numeric/multi-hop corpora, so pair it with a confidence-aware detector for safety-critical numeric answers.

---

## The two-input-channel problem (grounding ≠ injection defense)

RAG has **two** input channels: the user prompt **and** the retrieved context. Guarding only the user side is negligent — a study shows injecting documents into a guardrail's context flips its judgments in a meaningful fraction of cases, and your retrieved chunks are exactly such documents. **Indirect prompt injection** rides in through the retrieved channel.

> [!WARNING]
> **Grounding helps but does not eliminate injection.** Limiting responses to retrieved context reduces some injection, but indirect injection lives *inside* that context. You must validate the retrieved channel, enforce an instruction hierarchy (system > context > user), strip payloads at ingest, and constrain tool scope so retrieved content can't trigger privileged actions. Full detail in [09-enterprise-secure-rag](09-enterprise-secure-rag.md).

---

## Interview angles

- **"Debug a hallucination in production."** → localize: gold chunk retrieved or not? → retrieval fix vs grounding fix.
- **"How do you make answers accurate-or-admit-ignorance?"** → strict grounding instruction + faithfulness guardrail + "I don't know" path.
- **"Generation-time vs post-hoc citations?"** → bound-but-fallible vs verifiable-but-extra-pass; post-hoc correction improves accuracy.
- **"Is grounding enough to prevent prompt injection?"** → no — it helps but indirect injection rides in the retrieved context; layer guardrails + tool-scope limits.

---

## 📚 Resources

- 📄 [Hallucination Mitigation for RAG-LLMs](https://www.mdpi.com/2227-7390/13/5/856) — MDPI (2025) · paper · the mitigation taxonomy.
- 📄 [CiteFix: Post-hoc Citation](https://arxiv.org/html/2504.15629v2) — arXiv · post-hoc citation correction.
- 📄 [Generation-Time vs Post-hoc Citation (REASONS benchmark)](https://arxiv.org/html/2509.21557v2) — arXiv (Sep 2025) · the two-family comparison.
- 🎬 [Unlocking Advanced RAG: Citations & Attributions](https://www.youtube.com/watch?v=RnCuOL-LBAw) — video · practical citation patterns.
- 📘 [OWASP LLM Top 10](https://genai.owasp.org/llm-top-10/) — the injection threat-model checklist (LLM01 Prompt Injection).
