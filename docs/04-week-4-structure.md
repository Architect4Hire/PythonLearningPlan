# Week 4 — Structure and Storage

**17–21 August · Days 16–20**

**Goal:** close the `cross_cutting` and `trace` gap using the AST you're already parsing, then move to production storage.

This is the week your system stops being search over code and starts being retrieval that understands code structure. It's also the most differentiated work in the plan — almost nobody builds this, and it's what makes the difference on the questions developers actually ask.

---

## Day 16 — Mon 17 Aug · Prompt hardening and abstention

**What you're learning today:** the faithfulness failure mode that's specific to code, and how to measure refusal without rewarding a system that refuses everything.

### Hard block (2 hrs)

#### 1. The code-specific hallucination

The model knows a great deal of general C#. Ask it "how does the order repository handle concurrency?" and if your retrieved chunks don't cover it, the overwhelmingly likely output is a fluent, plausible description of how a typical EF Core repository handles concurrency — optimistic concurrency tokens, `DbUpdateConcurrencyException`, the usual.

It will be well-written, technically correct in general, and **wrong about your codebase**. This is worse than an obvious hallucination because it's unfalsifiable without checking, and it's the dominant faithfulness failure for code RAG.

The prompt has to attack this directly:

```
Answer only from the provided code excerpts.

Do not supplement with general knowledge of C#, .NET, or common library
behaviour. If the excerpts show a call to an external method whose body is
not provided, say that rather than describing what such a method typically does.

If the excerpts do not answer the question, state exactly what is missing —
which type or file you would need to see. Naming the gap is more useful than
a plausible guess.

Cite every claim as [n]. Reference types and members by their exact names
as they appear in the excerpts.
```

The "name the gap" instruction is doing something specific and useful: it converts a failure into a next action. "I can see `IOrderRepository.AddAsync` is called but the implementation isn't in the provided excerpts" tells the user exactly what to search for next, and it's honest.

#### 2. Two abstention metrics, not one

A system that refuses everything scores perfectly on refusing correctly. You need both directions:

```python
def abstention_metrics(results: list[CaseResult]) -> dict[str, float]:
    unanswerable = [r for r in results if r.case.kind == "unanswerable"]
    answerable   = [r for r in results if r.case.kind != "unanswerable"]
    return {
        "correct_refusal_rate": mean(r.refused for r in unanswerable),
        "over_refusal_rate":    mean(r.refused for r in answerable),
    }
```

Target: correct refusal above 0.8, over-refusal below 0.05. Detecting refusal needs a small classifier — a cheap LLM call with a yes/no prompt is fine, or a keyword heuristic if your prompt makes the model use a consistent phrase. The keyword approach is more brittle but free and deterministic, which matters given how often you'll run this.

#### 3. Ablate the prompt

Remove one instruction line at a time, run `make eval` on a 25-case subset each time. You're looking for which single line carries the weight.

It's rarely the one you'd guess. Frequently the "do not supplement with general knowledge" line does more than the explicit refusal instruction, because the failure isn't that the model won't say "I don't know" — it's that it doesn't notice it doesn't know.

#### 4. Re-tune k

You changed the funnel on Day 15 and you're changing the prompt now. The optimal number of chunks may have moved. Sweep `rerank_keep` ∈ {8, 12, 16, 20} with the new prompt and check faithfulness, not just recall.

More context is not monotonically better — this is where the lost-in-the-middle effect shows up in your own numbers.

### Consolidation (1–1.5 hrs)

1. **Read ten answers to unanswerable questions.** Are the refusals useful? "I don't know" is a failure; "the excerpts show `IPaymentGateway` is injected but its implementation isn't included — look in the Infrastructure project" is a success.
2. **Find an over-refusal** and diagnose it. Usually the retrieved context does contain the answer but indirectly, and the model is being too literal about what counts as "shown."
3. Lock in the prompt. Version it in `coderag/prompts.py` with a comment recording the ablation result.

### Reading (30 min)

Calibration and abstention in language models: why models are poorly calibrated about their own knowledge boundaries, and why retrieval doesn't automatically fix it.

### ✅ Done when

- Correct refusal > 0.8, over-refusal < 0.05
- Prompt ablation identifies the load-bearing instruction
- `rerank_keep` re-tuned against the new prompt

---

## Day 17 — Tue 18 Aug · The repo map

**What you're learning today:** that orientation is a different retrieval problem from lookup, and needs a different artifact.

### Hard block (2 hrs)

#### 1. The problem

Ask "where should I add a new order status?" and chunk retrieval struggles — there's no single chunk that answers it. The model needs to know what the codebase *contains* before it can reason about where something belongs.

The fix is a **repo map**: a compressed structural overview, always present in the prompt, that gives the model orientation the way a solution explorer gives you orientation.

#### 2. What goes in it

Budget roughly 1,500–2,500 tokens. Within that:

```
SOLUTION STRUCTURE
  src/Ordering.Domain          (42 files)  — entities, value objects
  src/Ordering.Application     (78 files)  — CQRS handlers, validators
  src/Ordering.Infrastructure  (56 files)  — EF Core, HTTP clients
  src/Ordering.Api             (31 files)  — controllers, DI wiring
  tests/Ordering.UnitTests     (94 files)

KEY TYPES
  Ordering.Domain.Orders
    Order : AggregateRoot
      Create(CustomerId, IReadOnlyList<OrderLine>) : Order
      AddLine(ProductId, int, Money) : void
      Confirm() : void
    OrderStatus : enum { Draft, Confirmed, Shipped, Cancelled }

  Ordering.Application.Orders
    CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
    IOrderRepository
      AddAsync(Order, CancellationToken) : Task
      GetAsync(Guid, CancellationToken) : Task<Order?>
```

Signatures only, no bodies. Folder descriptions from README files or from the most common type suffix in the folder.

#### 3. Selecting what's "key"

You can't fit every type. Rank by a cheap heuristic today (Day 19 improves it using the symbol graph):

| Signal | Weight |
|---|---|
| Public visibility | High — internal detail matters less for orientation |
| Is an interface or abstract type | High — these define the shape of the system |
| Is an aggregate root / entity / enum | High — domain vocabulary |
| Member count | Moderate — big types are usually important |
| Referenced from many files | High, but needs the graph — defer to Day 19 |
| In a test project | Exclude |

Build it at index time, cache it, invalidate on re-index. Regenerating per query is pure waste.

#### 4. Wire it in

Prepend to the system prompt, clearly delimited and labelled as orientation rather than evidence:

```
The following is a structural overview of the codebase, provided for
orientation. It lists signatures only. Do not cite it as a source —
cite only the numbered excerpts below.
```

Without that instruction the model will cite the repo map, and your citation-to-file mapping breaks.

#### 5. Measure

`make eval` with and without. Expect improvement on `cross_cutting` and on "where should X go" style questions, and no change on `symbol` questions. Also watch cost — you've added ~2k tokens to every single request, which is real money at eval-run volumes.

### Consolidation (1–1.5 hrs)

1. **Read the generated map end to end.** Would it orient *you* if you'd never seen this repo? That's the bar.
2. **Check the budget.** If it's over 2,500 tokens, tighten the selection rather than truncating — a truncated map cuts off alphabetically-late projects entirely.
3. **Try a deliberately architectural question** — "where would I add a new payment provider?" — before and after. This is the kind of question the map exists for.

### Reading (30 min)

Aider's repo map documentation and the reasoning behind it. Aider ranks with PageRank over a symbol graph, which is exactly where you're going on Day 19.

### ✅ Done when

- Repo map generated at index time, cached, under 2,500 tokens
- Wired in with an explicit no-cite instruction
- Measured; `cross_cutting` improved

---

## Day 18 — Wed 19 Aug · The symbol graph

**What you're learning today:** how to extract relationships from an AST, and how to be honest about the limits of syntactic analysis.

### Hard block (2 hrs)

#### 1. What you're building

A graph where nodes are symbols (types and members) and edges are relationships:

| Edge | Extracted from | Answers |
|---|---|---|
| `CALLS` | `invocation_expression` | "What calls `SendConfirmationEmail`?" |
| `IMPLEMENTS` | `base_list` on a type declaration | "Which classes implement `IOrderRepository`?" |
| `INHERITS` | `base_list` | "What derives from `AggregateRoot`?" |
| `INSTANTIATES` | `object_creation_expression` | "Where is `HttpClient` created directly?" |
| `REGISTERS` | `services.AddScoped<TIface, TImpl>()` | "What's the concrete type behind `IOrderRepository`?" |
| `TESTS` | Test file naming + references | "Where is this behaviour tested?" |

#### 2. tree-sitter queries

Manual traversal got you through Week 1. For relationship extraction, queries are much cleaner:

```python
CALLS_QUERY = CSHARP.query("""
(invocation_expression
  function: [
    (identifier) @callee
    (member_access_expression name: (identifier) @callee)
  ]) @call
""")

BASE_LIST_QUERY = CSHARP.query("""
(class_declaration
  name: (identifier) @type.name
  (base_list (_) @base)) @type.def
""")
```

Check whether your version returns captures as a dict or a list of tuples — this changed across releases and it's the first thing to verify in the REPL.

#### 3. Be honest about resolution

**tree-sitter is syntactic, not semantic.** This is a real limitation and pretending otherwise will cost you.

Given `_repository.AddAsync(order, ct)`, you can extract the callee name `AddAsync`. You cannot know that `_repository` is an `IOrderRepository` unless the field declaration is in the same file — and even then you're pattern-matching, not resolving types.

So your edges are **name-based and approximate**. If three types have an `AddAsync` method, a call to `AddAsync` produces three candidate edges.

Practical mitigations, in order of value:

1. **Same-file field type resolution.** Parse field and property declarations, build a local map of `_repository → IOrderRepository`, and use it to disambiguate receivers. Covers the majority of DI-style code and is genuinely worth building.
2. **Uniqueness bonus.** If a method name is unique across the repo, the edge is near-certain. Score edges by candidate count.
3. **Namespace proximity.** Prefer candidates in the same or a nearby namespace.
4. **Store confidence on the edge** and filter at query time.

Do not attempt full type resolution. That's a compiler, and Roslyn already exists — if you later need real semantic accuracy, the right answer is to shell out to a Roslyn-based analyser and merge its output, not to reimplement it in Python.

#### 4. Storage

SQLite. It's in the standard library, it handles this scale trivially, and recursive CTEs give you transitive queries for free:

```sql
CREATE TABLE symbols (
    id TEXT PRIMARY KEY, name TEXT, kind TEXT, file_path TEXT,
    namespace TEXT, parent_type TEXT, start_line INT, end_line INT
);
CREATE TABLE edges (
    src TEXT, dst TEXT, kind TEXT, confidence REAL, file_path TEXT, line INT
);
CREATE INDEX idx_edges_dst ON edges(dst, kind);
CREATE INDEX idx_edges_src ON edges(src, kind);
CREATE INDEX idx_symbols_name ON symbols(name);
```

Build it during indexing, from the same parse you're already doing. No extra parsing pass.

#### 5. Query helpers

```python
def callers_of(symbol: str, min_confidence: float = 0.5) -> list[Symbol]: ...
def implementations_of(interface: str) -> list[Symbol]: ...
def registered_impl(interface: str) -> Symbol | None: ...
def tests_for(symbol: str) -> list[Symbol]: ...
def call_path(src: str, dst: str, max_depth: int = 4) -> list[list[Symbol]]: ...
```

`call_path` uses a recursive CTE and answers "how does a request get from the controller to the database?" — one of the more impressive things this system will do.

### Consolidation (1–1.5 hrs)

1. **Spot-check twenty edges** against the actual source. Compute a rough precision figure. Below 0.8 means your resolution heuristics need work before Day 19 builds on them.
2. **Graph statistics.** Node count, edge count by kind, and the distribution of in-degree. The highest in-degree symbols are your most-called code — check whether they match your intuition about the codebase. If something surprising is at the top, either you've learned something or you have a bug.
3. **Find the orphans.** Public symbols with zero inbound edges are either dead code or called via reflection/DI in a way you're not capturing. Both are interesting.

### Reading (30 min)

Static analysis fundamentals: call graphs, the distinction between syntactic and semantic analysis, and why dynamic dispatch makes precise call graphs undecidable in general. This is why your confidence scores exist.

### ✅ Done when

- Graph built during indexing with edge precision above 0.8 on spot-checks
- All six query helpers working
- `call_path` returns sensible paths on a known flow

---

## Day 19 — Thu 20 Aug · Graph-augmented retrieval

**What you're learning today:** how to combine similarity retrieval with structural expansion, and how to measure a change that only affects some question kinds.

### Hard block (2 hrs)

#### 1. The pattern

Similarity retrieval finds chunks that *look like* the question. Graph expansion adds chunks that are *structurally related* to what was found.

```
query
  → hybrid retrieve 50
  → rerank to 12                    ← the "seed" set
  → graph-expand along edges        ← add related chunks
  → budget-cap to ~16 total
  → generate
```

The seed set stays authoritative. Expansion supplements it.

#### 2. Expansion rules

Different question kinds want different edges, and this is where the taxonomy from Week 2 pays off:

| Seed chunk is | Add | Why |
|---|---|---|
| A method | Its callers (top 3 by confidence) | "How is this used?" |
| A method | Its direct callees, if defined in-repo | "What does this actually do?" |
| A class implementing an interface | The interface definition | Contract and doc comments |
| An interface | Its implementations (top 3) | "Which class actually does this?" |
| Any symbol | Its test, if one exists | Tests document intent |
| An interface | Its DI registration | "What's the concrete type?" |

Cap the expansion. A method with 40 callers must not flood the context — take the top few by confidence and note the total count in the header: `// 40 callers, showing 3`. That count is itself useful information for the model.

#### 3. Query-aware expansion

Better than expanding uniformly: let the question shape the expansion. A lightweight classifier — keyword heuristics are fine, or a cheap LLM call — routes to a strategy:

- Question contains "calls", "used", "who invokes" → weight `CALLS` edges heavily
- Question contains "implements", "which class", "concrete" → weight `IMPLEMENTS` and `REGISTERS`
- Question contains "test", "how do I use" → weight `TESTS`
- Otherwise → light uniform expansion

Keep the heuristic version. It's free, deterministic, and testable, and the LLM classifier adds latency for a small gain.

#### 4. Rank the repo map with PageRank

Now that the graph exists, improve Day 17's heuristic ranking. Run PageRank over the `CALLS` graph — heavily-called symbols are structurally central, which is a much better relevance signal than member count.

```bash
uv add networkx
```

Regenerate the map with PageRank ordering and compare. The map should now surface the types that actually matter in this codebase rather than the biggest ones.

#### 5. Measure

Run `make retrieval-eval` and look specifically at the two kinds this was built for:

```
kind            n   before   after
cross_cutting  10    0.30  →  0.80
trace           8    0.25  →  0.75
symbol         12    0.92  →  0.92
behaviour      12    0.71  →  0.74
```

Something like this is the expected shape. If `cross_cutting` and `trace` don't move substantially, either your edges are wrong (check Day 18's precision figure) or the expansion isn't reaching the final context (check the trace).

**Watch for regressions on the kinds that were already good.** Expansion adds chunks, which can push a correct seed chunk down the context. If `symbol` drops, tighten the expansion budget.

### Consolidation (1–1.5 hrs)

1. **Re-run the Week 1 Day 5 failures.** Those `cross_cutting` and `trace` questions that failed badly on day five — most should now work. This is the most satisfying moment in the plan; note it for the write-up.
2. **Trace one expanded query end to end.** Confirm the seed chunks, the expansion, and the final context are what you expect.
3. **Update `ARCHITECTURE.md`** with the graph layer.

### Reading (30 min)

Graph-based RAG approaches — GraphRAG and LightRAG. Note that both build graphs of *extracted entities* from unstructured prose using an LLM, which is expensive and error-prone. You get a real graph from the AST for free, because code has explicit structure that prose doesn't. That's a genuine advantage of this domain and worth saying in your write-up.

### ✅ Done when

- `cross_cutting` and `trace` substantially improved
- No regression on `symbol` or `behaviour`
- Repo map ranked by PageRank
- The Day 5 failures now largely pass

---

## Day 20 — Fri 21 Aug · Qdrant and metadata filtering

**What you're learning today:** what a production vector database gives you, and what ANN parameters actually cost.

### Hard block (2 hrs)

#### 1. Stand up Qdrant

```bash
docker run -p 6333:6333 -p 6334:6334 \
  -v $(pwd)/qdrant_storage:/qdrant/storage \
  qdrant/qdrant
```

Dashboard at `localhost:6333/dashboard`. Being able to see your collections and inspect payloads is worth the migration on its own.

#### 2. Implement `QdrantStore`

```bash
uv add qdrant-client
```

Satisfy the existing `VectorStore` protocol. Config switch: `CODERAG_VECTOR_STORE=chroma|qdrant`.

Points to watch:
- **Distance metric** must be `COSINE`, matching Chroma's configuration from Day 4
- **Payload indexes** on `file_path`, `project`, `is_test`, `kind`, `language` — without these, filtering scans
- **Batch upserts** of 256, same as before

#### 3. Verify parity

Re-index and run `make retrieval-eval`. **Scores should match Chroma within noise.**

If they don't, you have a bug, and there are only a few candidates:
- Distance metric mismatch (L2 versus cosine)
- Vectors not normalised on one path
- Payload field name mismatch breaking your `file_path` extraction
- Different `ef_search` default changing recall

Find it before moving on. A silent parity failure here would invalidate every measurement you make afterwards.

#### 4. Metadata filtering end to end

```bash
coderag ask "how are orders validated?" --no-test
coderag ask "..." --project Ordering.Application
coderag ask "..." --kind method
```

Two things to be careful about:

**Filter before search, not after.** Post-filtering means asking for 50 results, discarding 40 as tests, and being left with 10. Qdrant's payload filters apply during the HNSW traversal.

**Aggressive filters degrade ANN recall.** When a filter excludes most of the corpus, HNSW's graph traversal has fewer valid neighbours to walk through and can fail to reach the relevant region. Qdrant handles this by falling back to exact search below a cardinality threshold, but be aware of the effect and test a heavily-filtered query.

#### 5. The HNSW curve

Sweep `ef_search` ∈ {16, 32, 64, 128, 256} and measure recall@10 against query latency. Plot it.

You'll find a knee — a point past which more search effort buys almost nothing. At your corpus size, defaults are almost certainly fine, but having generated this curve yourself means "approximate nearest neighbour" is now a concrete tradeoff rather than a phrase.

Also try `m` ∈ {8, 16, 32}, which requires a rebuild. Higher `m` means better recall and more memory.

### Consolidation (1–1.5 hrs)

**Write-up #4:** *"Answering 'what calls this?' — adding a symbol graph to code RAG."*

This is your best post. The before/after on `cross_cutting` and `trace` is dramatic, the reason similarity search structurally cannot answer those questions is genuinely interesting, and the contrast with LLM-extracted graphs in GraphRAG is a real insight about the domain.

Then update `ARCHITECTURE.md` and commit the HNSW curve.

### Reading (30 min)

Malkov & Yashunin on HNSW. This algorithm sits under every vector database you will ever use, and you now have your own measurements to read it against.

### ✅ Done when

- Qdrant parity confirmed against Chroma
- Filtering works end to end, applied pre-search
- HNSW recall/latency curve plotted
- Write-up published

---

## Week 4 retrospective

You should be able to explain:

- Why code RAG's dominant hallucination is fluent general C# knowledge, and how the prompt attacks it
- Why orientation and lookup are different retrieval problems needing different artifacts
- What tree-sitter can and cannot resolve, and why your edges carry confidence scores
- Why an AST-derived graph is more reliable than an LLM-extracted one
- What `ef_search` trades off, from your own curve

**What you've built:** a system that answers structural questions about a codebase, not just similarity questions. This is the part that would be hard for someone else to replicate.

**What's left:** it only runs on your machine, from a CLI, with no tests. Week 5 makes it a service.
