# RAG Evaluation

> *Updated 2026-07 · part of [RAG Engineer Interview Questions](../README.md) · drill the [evaluation question bank](../questions/evaluation.md)*

Every RAG demo looks great on the three questions you tried. Evaluation is how you know it works on the *next thousand* — and how you catch a "small" prompt tweak or a silent vendor embedding update that quietly breaks retrieval. **"Eval is the new system design"** in 2026, and interviewers know it: anyone can wire a retriever and a prompt; the senior engineer can say "here is how I'd know it works, here is where my metrics lie, and here is how I gate a deploy so a silent regression never reaches users."

The cardinal mistake is grading the system as **one black box**. When a query fails you won't know if retrieval missed, the reranker cut the right chunk, or the generator ignored good context.

---

## The retrieval (IR) metrics — define and choose

| Metric | Rewards | Use when |
|---|---|---|
| **Recall@k** | gold chunk anywhere in top-k | first-stage **ceiling** (must-have) — a miss is unrecoverable |
| **Hit rate@k** | single gold doc retrieved (Y/N) | one correct doc per query |
| **MRR** | the *first* relevant hit, high (`1/rank`) | one right answer, position matters |
| **nDCG@k** | graded relevance, position-discounted | multiple relevant chunks; **rerank quality** |
| **Precision@k** | share of top-k that's relevant | context noise / token budget |

The senior move: **pick the metric to match the task.** `recall@k` is the must-pass ceiling for the retriever; `nDCG@k` (or `MRR` for single-answer queries) judges whether the reranker orders well. Tie each metric to the failure it exposes — that's the signal.

```python
# The two IR metrics every RAG eval starts with -- on a labelled set.
def recall_at_k(eval_set, retrieve, k=20):
    hits = 0
    for query, gold_chunk_ids in eval_set:
        got = {r["chunk_id"] for r in retrieve(query, k=k)}
        hits += 1 if got & set(gold_chunk_ids) else 0   # did ANY gold chunk make top-k?
    return hits / len(eval_set)

def precision_at_k(eval_set, retrieve, k=5):
    total = 0.0
    for query, gold_chunk_ids in eval_set:
        got = [r["chunk_id"] for r in retrieve(query, k=k)]
        total += len(set(got) & set(gold_chunk_ids)) / k   # how clean is top-k?
    return total / len(eval_set)
# Rule of thumb: maximize recall@50-100 in stage 1, buy precision@5 with a reranker.
```

---

## Split the eval: retrieval vs generation (the 2×2)

Instrument every stage and run two families. **Retrieval:** context precision (was the context relevant?) and context recall (did you fetch all the needed context?), independent of the generator. **Generation:** faithfulness/groundedness (every claim supported by context?) and answer relevancy (does it address the question?), with retrieval fixed.

|  | Generation good | Generation bad |
|---|---|---|
| **Retrieval good** | ✅ working | model ignored / hallucinated past good context → fix grounding prompt / faithfulness |
| **Retrieval bad** | faithfully answered from *wrong* context → fix chunking / hybrid / rerank | both bad → start with retrieval (generation can't exceed its context) |

A real case: end-to-end accuracy *dropped* after a chunking change, but splitting the metrics showed `recall@5` actually **rose** while precision collapsed — a parser was fragmenting tables. Without the split, the team would have reverted a real improvement.

> [!TIP]
> **"Your answers are wrong — how do you debug?"** → split the eval, locate the failing stage with the 2×2, then name the lever. That structure alone outperforms most candidates.

---

## Where the metrics lie

Treat each metric as one alarm, not truth. The most-cited example: default **Ragas faithfulness returns *null*** (not a low score) on a large share of hard prompts, because it decomposes the answer into atomic claims and checks *entailment* — and on numeric/multi-step answers the extraction step itself fails:

| Corpus | Ragas faithfulness "no output" rate |
|---|---|
| FinanceBench (numeric, multi-step) | **83.5%** ← entailment breaks here |
| DROP (discrete reasoning) | 58.9% |
| CovidQA | 21.2% |
| PubMedQA | 0.1% |

> [!WARNING]
> **Never ship a RAG safety story on faithfulness alone for numeric / multi-hop corpora.** A *null* is not a pass. Any metric is one alarm with a known blind spot: faithfulness is blind to numeric/multi-hop, answer-relevancy says nothing about correctness, recall says nothing about whether the model *used* what you fetched. Triangulate, and add a confidence-aware hallucination detector for safety-critical numeric corpora.

```python
from ragas import evaluate
from ragas.metrics import faithfulness, answer_relevancy, context_precision, context_recall
# rows: question, answer, contexts, ground_truth
result = evaluate(dataset, metrics=[context_precision, context_recall,   # retrieval
                                    faithfulness, answer_relevancy])     # generation
# per-metric scores you can gate a deploy on -- but read the caveats above
```

---

## The deploy gate: metrics that block a bad ship

Metrics only protect you if they *block a bad deploy*. Freeze a **golden set**, define per-metric thresholds, and run the eval in CI on every prompt / model / chunker / index change — failing the build if any metric regresses past a margin against the frozen baseline. Pin model and prompt versions alongside the gate so drift is attributable.

---

## Golden datasets & error analysis

The discipline that actually moves quality (Hamel Husain / Shreya Shankar): **open-code** real production traces (read + label failures), group into a taxonomy (**axial coding**), stop at saturation. Prefer **binary pass/fail** over 1–5 Likert — reviewers regress to "3" and it means nothing across annotators.

> [!WARNING]
> **The 0.91-vs-0.40 trap.** One team read 0.91 recall offline and 0.40 in production because their gold *contexts* came from hand-picked chunks the live retriever never surfaced — the offline eval graded a corpus the system never uses. The rule: **generate gold questions, but harvest the gold contexts from real production retrieval runs.** "I'd have an LLM generate Q&A pairs" is the answer that hides the 0.40.

---

## LLM-as-judge: biases & calibration

LLM judges are cheap, indispensable, and quietly biased: **position bias** (GPT-4o picks the first of two identical responses ~64% of the time), **verbosity bias** (longer scores higher), **self-preference** (models favor their own family). Mitigations: swap positions and average; multi-judge ensemble for high-stakes gates; a calibration loop — hand-label a gray-zone set, fold corrections into the judge prompt as few-shot, and **track agreement with Cohen's κ** (one team went 62% → 0.78).

> [!WARNING]
> **The judge is another model output that itself needs evaluating.** "I trust GPT-4 to grade" without validating against human labels is the answer that fails. Report κ (chance-corrected — raw % agreement flatters), iterate the judge prompt until κ clears ~0.7, and prefer narrow binary questions ("is every claim supported by the context? yes/no") over a vague 1–10 score.

---

## Offline vs online eval

- **Offline:** the CI gate on a frozen golden set — catches regressions before ship.
- **Online:** shadow/canary on real production traffic, watching per-span `hit@k` on the *long tail*. Offline sets over-sample easy queries and can be flattered by a new vendor's own tuning; the long tail can still collapse. **Embedding swaps are exactly the silent-drift case** — only the production distribution + per-span tracing reveals long-tail recall loss.

---

## Interview angles

- **"Which retrieval metric would you report and why?"** → `recall@k` as the must-pass ceiling; `nDCG@k`/`MRR` for ranking quality.
- **"Evaluate a RAG pipeline end-to-end?"** → split retrieval vs generation, the 2×2, gate in CI, then online canary.
- **"What is faithfulness in Ragas vs context precision/recall?"** → claim-entailment vs context relevance/coverage; and where faithfulness nulls out.
- **"How do you build a golden set?"** → error-analysis on real traces, binary labels, gold contexts from the *production retriever*.
- **"How do you know your LLM judge is any good?"** → Cohen's κ against hand-labelled ground truth; calibrate the prompt.
- **"When Ragas vs DeepEval vs TruLens?"** → Ragas (RAG-specific metrics), DeepEval (15+ benchmarks, CI-friendly), TruLens (tracking + groundtruth).

---

## 📚 Resources

- 💻 [explodinggradients/ragas](https://github.com/explodinggradients/ragas) — **14.6k★, Apache-2.0** · RAG-specific: faithfulness, context precision/recall, answer relevancy.
- 💻 [confident-ai/deepeval](https://github.com/confident-ai/deepeval) — **16.6k★, Apache-2.0** · 15+ benchmarks; agentic/chatbot/RAG metrics; CI-friendly.
- 💻 [truera/trulens](https://github.com/truera/trulens) — **3.4k★, MIT** · eval + tracking; groundtruth-based.
- 📘 [Ragas: Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/) · [Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/) — the metric definitions.
- 📄 [Evaluation Metrics for Search / Rec Systems](https://weaviate.io/blog/retrieval-evaluation-metrics) — Weaviate · precision, recall, MRR, MAP, nDCG.
- 📄 [DeepEval vs Ragas](https://deepeval.com/blog/deepeval-vs-ragas) · [DeepEval vs TruLens](https://deepeval.com/blog/deepeval-vs-trulens) — the framework tradeoffs.
- 📄 [RAG Eval Tools: W&B vs Ragas vs DeepEval vs TruLens vs UpTrain](https://aimultiple.com/rag-evaluation-tools) — AImultiple (Mar 2026) · the roundup.
- 🎬 [Why AI Evals Are the Hottest New Skill](https://youtube.com/watch?v=BsWxPI9UM4c) — Hamel Husain · the "evals are the highest-leverage skill" argument.
