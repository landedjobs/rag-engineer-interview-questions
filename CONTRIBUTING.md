# Contributing

Thanks for helping build the best RAG-engineer interview-prep on GitHub. This repo has a **quality bar**, not just a style guide — contributions that clear it get merged fast.

## What we want

- **Real interview questions** — with a provenance label (see below).
- **Multiple-choice questions** — with a *why* for every option (the wrong options must teach).
- **Typed, annotated resources** — never a bare link.
- **Worked system designs** — scored against [the rubric](answers/rag-system-design-rubric.md).
- **Corrections** — outdated numbers, dead links, better explanations. Dated content is a feature; keep it current.

## The quality bar (non-negotiable)

1. **Answer-first, senior voice.** Lead with the mechanism, then the production implication. No fluff, no marketing.
2. **Every claim is defensible.** Prefer a metric or a cited source over an assertion. "Use a bigger model" is never the answer.
3. **Provenance labels on real questions:**
   - **✅ Reported** — attested in a public source. Include the link.
   - **🔮 Representative** — realistic and in-style, but not attributed. Mark it honestly.
4. **Typed links.** Tag every link: `📄 paper · 📘 docs · 🎬 video · 🧑‍🏫 course · 🛠️ tool · 💻 repo`. Annotate what it is and why it's here. Cap ~8 per topic — curate, don't dump.
5. **License-note repos.** State stars + license, and only adapt MIT / Apache-2.0 / CC0 code (with attribution). CC-BY-SA and research-artifact licenses are **link + commentary only**.
6. **Dated 2026.** Version claims to the current landscape (GraphRAG, agentic RAG, contextual retrieval, eval-as-system-design, secure RAG).

## Question format

```markdown
**Q · 🟡 [The scenario — concrete, a real decision].**

- ▫️ [plausible wrong option] — *[why it's wrong, in one sentence]*
- ✅ [correct option] — *[why it's right, tied to a mechanism or metric]*
- ▫️ [another wrong option] — *[why it's wrong]*
```

Difficulty tiers: 🟢 Core · 🟡 Senior · 🔴 Staff. Put the question in the right [topic file](questions/).

## Resource format

```markdown
- 📄 [Title](https://url) — Source (date) · one-line annotation: what it is + why it's here.
- 💻 [owner/repo](https://github.com/owner/repo) — **N★, LICENSE** · what to mine it for.
```

## Workflow

1. Open an issue first for anything large (use the [templates](.github/ISSUE_TEMPLATE/)).
2. One topic per PR keeps review fast.
3. Run a link check locally if you can (`lychee`), or let CI catch it.
4. Fill in the [PR template](.github/PULL_REQUEST_TEMPLATE.md).

By contributing you agree your contributions are licensed under the repo's [MIT License](LICENSE).
