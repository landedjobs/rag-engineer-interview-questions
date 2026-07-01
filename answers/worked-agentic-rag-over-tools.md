# Worked Design — Agentic RAG over Tools

> *Scored against [the RAG system-design rubric](rag-system-design-rubric.md) · pair with [06-advanced-rag](../content/06-advanced-rag.md)*

**Prompt:** "Design an assistant that answers multi-step research questions over an internal knowledge base *and* live tools (a SQL warehouse, a ticketing API, web search). It should decide what to retrieve, reason over intermediate results, and re-retrieve as needed."

---

## Step 0 — Clarify

- **Shape?** Mixed: closed-corpus retrieval **+** tool use. Genuinely multi-hop → agentic is justified here (unlike single-hop lookups).
- **Scale / QPS?** Lower QPS, higher per-query cost tolerance → the 4–7× latency of an agent loop is acceptable.
- **Stakes?** Tools have side effects (SQL, ticketing) → **tool safety and least-privilege** are first-class.
- **Freshness?** Live tools are always-fresh; the KB needs CDC.

> **The senior caveat first:** agentic RAG is the *wrong* tool for single-hop lookups — it multiplies latency and can loop-stuck. State that you'd route simple queries to a plain retrieve-then-answer path and reserve the agent loop for genuine multi-step questions.

---

## The loop — with a hard stop

Pattern: **retrieve → reason → decide (answer / re-retrieve / call a tool) → repeat**, bounded by a step and budget cap. Ground it in the two foundational patterns:

- **Self-RAG** — reflection tokens decide *when* to retrieve and whether passages are relevant/supported.
- **CRAG (Corrective RAG)** — a lightweight evaluator grades retrieval; on low confidence, trigger a web search or query rewrite instead of answering from bad context.

```python
def agentic_rag(query, tools, max_steps=4, budget_tokens=8000):
    scratchpad, spent = [], 0
    for step in range(max_steps):                       # hard stop, not an infinite loop
        ctx = retrieve(query, scratchpad)
        if grade_relevance(query, ctx) == "sufficient": # CRAG-style confidence gate
            return generate(query, ctx, scratchpad, cite=True)
        action = plan_next(query, ctx, scratchpad, tools)  # re-retrieve OR call a scoped tool
        result, spent = run_scoped(action, tools, spent, budget_tokens)  # least-privilege tools
        scratchpad.append(result)
    return generate(query, ctx, scratchpad, note="low-confidence", cite=True)  # stop condition
```

---

## Tool safety (the part that separates senior answers)

- **Least-privilege scope:** each tool token grants exactly what this user may touch. Retrieved content must never be able to steer a privilege-bearing call (indirect prompt injection defense).
- **Read vs write separation:** SQL is read-only by default; any write/ticket-create action requires an explicit confirmation step or human-in-the-loop. *"Agent deleted a prod table"* is a real failure class — irreversible actions get a guardrail.
- **Whitelist tool calls;** validate tool arguments against a schema before execution.

---

## Evaluation — the 3-metric agentic stack

A single end-to-end score hides *where* the trajectory failed. Use the 2026 agentic-eval stack:

| Metric family | What it measures |
|---|---|
| **Outcome** | Task Success Rate — did it answer correctly? |
| **Trajectory** | steps / tokens per success, Tool Call Accuracy — was the path efficient and correct? |
| **Tool / reasoning** | schema compliance, reasoning soundness — did each step make sense? |

Static benchmarks measure the model; **dynamic trajectory benchmarks measure the system.** Calibrate any LLM judge against human labels (Cohen's κ) and use a **cross-family judge** (a judge sharing the agent's model family drifts toward self-preference).

---

## Silent failures

- **Loop-stuck / runaway cost** → max-steps + token budget + confidence gate.
- **Wrong tool selection** → improve tool descriptions, few-shot the tool-choice, and track Tool Call Accuracy.
- **Injection via retrieved content steering a tool** → instruction hierarchy + least-privilege scope + argument validation.
- **Latency blowup on simple queries** → route single-hop queries to the non-agentic path.

---

## Rubric scorecard

| Stage | Decision |
|---|---|
| Framing | multi-hop KB + live tools; route single-hop to plain RAG |
| Loop | retrieve → reason → decide, Self-RAG/CRAG confidence gate, hard stop |
| Tool safety | least-privilege scope, read/write separation, HITL for irreversible actions |
| Retrieval | hybrid + rerank per hop; ground-and-cite the final answer |
| Eval | outcome + trajectory + tool/reasoning; cross-family judge, κ-calibrated |
| Ops | trace every step; cap steps/tokens; monitor Tool Call Accuracy |
