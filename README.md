# Building a Code RAG System — A Five-Week Plan

A structured, measurement-driven plan for building a retrieval-augmented generation system over a real C# codebase, from first principles.

Written for experienced engineers coming to the Python AI ecosystem from another stack. It assumes you can read and write code well, and assumes nothing about embeddings, vector search, or retrieval.

---

## What you'll build

A system that answers questions about your own codebase with citations to real files and line numbers:

- **AST-aware chunking** with tree-sitter — chunks are methods and types, not arbitrary token windows
- **Hybrid retrieval** — code-specific embeddings fused with identifier-aware BM25
- **Cross-encoder reranking** with a tuned candidate funnel
- **A symbol graph** extracted from the AST, answering "what calls this?" and "which classes implement this?"
- **Agentic multi-hop retrieval** for questions requiring sequential lookups
- **A 60-case evaluation harness** with objective, LLM-free retrieval metrics
- **FastAPI service** with SSE streaming, containerised, tested, in CI

Every design decision is validated against measurements rather than asserted.

---

## Why code RAG is not document RAG

Most RAG material assumes a prose corpus. A codebase breaks those assumptions in specific ways, and the plan is built around them:

| Assumption | Why it fails for code |
|---|---|
| Fixed-size chunks are fine | Slicing every 512 tokens cuts through the middle of methods |
| Chunks are self-describing | A method body contains none of the words someone would search for |
| Embeddings are the primary retriever | Developers search for exact identifiers; BM25 often dominates |
| Standard tokenisation works | `CreateOrderCommandHandler` is one token, so "order handler" matches nothing |
| Ground truth needs an LLM judge | For code it's a file path — exact, free, instant |
| Similarity search answers the question | "What calls this?" is not textually similar to its answer |

The primer covers all seven in detail.

---

## The documents

Read in order. The primer is theory; each week file is a self-contained five-day plan.

| File | Covers |
|---|---|
| [`00-code-rag-primer.md`](docs/00-code-rag-primer.md) | Concepts from zero — embeddings, ANN, BM25, cross-encoders, context windows. Why code RAG differs. Stack decisions. |
| [`01-week-1-ingestion.md`](docs/01-week-1-ingestion.md) | Repo walking, tree-sitter parsing, symbol chunking with context headers, embedding, first working system |
| [`02-week-2-evaluation.md`](docs/02-week-2-evaluation.md) | Package structure, tracing, citations, 60-case golden set, objective retrieval metrics, baseline |
| [`03-week-3-retrieval.md`](docs/03-week-3-retrieval.md) | Chunking ablations, embedding bake-off, identifier-aware BM25, rank fusion, reranking |
| [`04-week-4-structure.md`](docs/04-week-4-structure.md) | Prompt hardening, repo map, symbol graph, graph-augmented retrieval, Qdrant |
| [`05-week-5-capstone.md`](docs/05-week-5-capstone.md) | Git-aware incremental indexing, FastAPI + SSE, tests and CI, containers, agentic retrieval, capstone |

---

## Structure

Twenty-five days, Monday to Friday, three to four hours each. Roughly 90 hours total.

| Week | Theme | End state |
|---|---|---|
| 1 | Ingestion | AST-chunked index of your repo; working answers by Friday |
| 2 | Instrumentation & evaluation | Traced, packaged, 60-case golden set, recorded baseline |
| 3 | Retrieval quality | Hybrid search and reranking, every Week 1 decision measured |
| 4 | Structure & storage | Symbol graph closing the `cross_cutting` and `trace` gap; Qdrant |
| 5 | Serving & capstone | Streaming API, tests, CI, containers, agentic retrieval |

### The daily shape

| Block | Time | Mode |
|---|---|---|
| Hard block | 2 hrs, morning | New concept plus build. Peak focus. |
| Consolidation | 1–1.5 hrs | Refactor, inspect traces, read library source. Low load, high retention. |
| Reading | 30 min | Papers. No coding. |

Two hours is a ceiling on genuinely new material, not a target to beat. Every day ends with a **Done when** check — don't start the next day until it passes.

---

## Prerequisites

- **A codebase you know well.** 50k–500k lines, real rather than sample. You must be able to spot a wrong answer instantly; this outweighs every other selection criterion.
- Python 3.12 and `uv` on Linux, macOS, or WSL2
- An Anthropic API key (budget $20–30 across five weeks)
- Docker, from Week 4
- No GPU required. Everything runs on CPU; a GPU makes indexing faster and changes nothing else.

---

## Principles

**Build the walking skeleton first.** A working end-to-end system by day four, however crude. Correctness after.

**Evaluation before optimisation.** The golden set and metrics harness are built in Week 2, before any tuning. Weeks 3 and 4 are measured; nothing before them was.

**Never keep a change that didn't move a number.** Every experiment ends with an eval run. The threshold is 0.02 on recall@10. Sensible-sounding changes get reverted like any other.

**Implement before importing.** Cosine similarity, reciprocal rank fusion, and BM25 tokenisation are written by hand before any library is reached for. They're ten lines each and the understanding is the point.

**Read the source.** RAGAS's faithfulness metric, LlamaIndex's node parsers, HNSW parameters. Consolidation blocks exist for this.

---

## The measurement that makes this work

For a prose corpus, "was the right context retrieved?" requires a judgement call, usually delegated to an LLM judge — slow, expensive, and noisy.

For code, ground truth is a **file path**. That makes `recall@k`, `hit rate@k`, and `MRR` exact, free, deterministic, and instant. The full 60-case suite runs in ten seconds and costs nothing.

This changes the working rhythm entirely. You run it after every change, dozens of times a day, and reserve LLM-judged metrics for daily confirmation. It's the single largest practical advantage of this domain, and Week 2 is built around it.

---

## Adapting this

**Different language?** Swap the tree-sitter grammar and the node type sets in Week 1. Everything downstream is language-agnostic. The C#-specific parts are the declaration node names, the `<auto-generated>` detection, and the interface `I`-prefix handling in tokenisation.

**Less time?** A one-hour-per-day version of this plan runs about ten weeks. Cut in this order: agentic retrieval (Day 24), query transformation, structural chunking refinements. Never cut Week 2 — the golden set and eval harness are what separate this from a tutorial.

**Different corpus?** The structural retrieval in Week 4 depends on code having explicit relationships. For prose, that week becomes conventional RAG work and the objective file-path metric no longer applies.

---

## A note on the write-ups

Each week ends with a public post. Five in total, roughly 45 minutes each.

They're in the plan for two reasons. Explaining a result is the fastest way to find out whether you understood it. And a build log with real numbers — including the parts that didn't work — is a more legible demonstration of capability than a finished repo alone.

The unflattering baseline post at the end of Week 2 is usually the most read.
