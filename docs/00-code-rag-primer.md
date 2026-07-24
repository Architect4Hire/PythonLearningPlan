# Code RAG — Concepts Primer

**Read this before Monday.** It's the theory floor for the five weeks, plus the reasoning behind every stack decision. Roughly 45 minutes.

---

## Part 1 — What RAG actually is

A language model has two sources of knowledge: what was baked into its weights during training, and whatever you put in the prompt. That's it. There is no third place.

Retrieval-Augmented Generation is the second one, done deliberately: when a question arrives, go find the most relevant text you have, paste it into the prompt, and ask the model to answer using it.

That's the whole idea. Everything else in these five weeks is engineering around one problem: **the prompt is small and your codebase is large.**

A C# analogy that holds up reasonably well: the model is a very capable contractor who has read a great deal of general C# but has never seen your solution. RAG is you handing them the four files relevant to the ticket. Fine-tuning would be sending them on a training course. Almost always, handing over the right files is the correct move — it's cheaper, it updates instantly when the code changes, and you can point at exactly which file produced the answer.

**What RAG is not:**
- Not fine-tuning. You aren't changing the model.
- Not memory. Each request is independent; you re-retrieve every time.
- Not search with extra steps — though a bad RAG system is exactly that, and knowing the difference is most of the skill.

### The pipeline

Two phases, and they run at completely different times.

**Indexing** (offline, occasionally):
```
source files → parse → chunk → embed → store in vector index
```

**Querying** (online, every request):
```
question → embed → search index → rerank → build prompt → generate → answer + citations
```

The asymmetry matters. Indexing can take twenty minutes; querying must take two seconds. Almost every design tradeoff in this plan is about moving work from the query path to the index path.

### The three failure classes

You will use these constantly. Every bad answer is one of exactly three things, and telling them apart requires looking at *what was retrieved*, not just the answer:

| Failure | What happened | Where you fix it |
|---|---|---|
| **Recall** | The right chunk was never retrieved | Chunking, embeddings, hybrid search |
| **Precision** | Junk was retrieved and crowded out the good chunk | Reranking, filtering, k tuning |
| **Faithfulness** | The right chunk *was* retrieved and the answer is still wrong | Prompting, context formatting |

Learn to classify a failure in thirty seconds by opening the trace. This single habit is worth more than any individual technique in the plan.

---

## Part 2 — Five concepts you need to own

### 1. Embeddings

An embedding model maps a piece of text to a fixed-length array of floats — typically 384, 768, or 1024 of them. The property that makes it useful: **texts with similar meaning land near each other in that space**, even when they share no words.

```python
embed("calculate order total")     # → [0.021, -0.083, 0.044, ...]  768 floats
embed("compute the sum of a cart") # → nearby vector
embed("configure SMTP settings")   # → far away
```

"Near" is measured by cosine similarity — the cosine of the angle between two vectors, ranging from -1 to 1. If vectors are normalised to unit length (they usually are), this is just a dot product.

$$\text{cos}(a,b) = \frac{a \cdot b}{\|a\| \|b\|}$$

You'll implement this by hand on Day 4. It's three lines and it makes the whole thing concrete.

**The critical caveat for you:** embedding models are trained on data, and most are trained overwhelmingly on prose. A model that's excellent at "these two paragraphs mean the same thing" can be mediocre at "these two methods do the same thing." This is why the stack below uses a code-specific embedding model, and why Week 3 has you benchmark it against a general one on your own repo.

### 2. Vector indexes and approximate search

You have 8,000 chunks. A query arrives. Comparing against all 8,000 vectors is fine — brute force at that scale takes milliseconds. At 8 million it isn't.

Vector databases solve this with **approximate nearest neighbour** search. The dominant algorithm is HNSW (Hierarchical Navigable Small World): a layered graph where each node links to nearby nodes, with sparse long-range links in upper layers. Search descends from the top layer, greedily walking toward the query.

The word to notice is *approximate*. HNSW can miss true nearest neighbours. Two parameters control the tradeoff:

- **`m`** — links per node. Higher means better recall, more memory.
- **`ef_search`** — how many candidates to keep during the walk. Higher means better recall, slower queries.

You'll plot this curve yourself in Week 4. For your scale, defaults will be fine, but understanding that your retrieval is *approximate* explains a class of otherwise baffling bugs.

### 3. Lexical retrieval and BM25

Before embeddings, search worked on word overlap. BM25 is the refined version of that idea and it is still, in 2026, extremely hard to beat on certain queries.

It scores a document by how many query terms it contains, weighted by:
- **Rarity** — matching a rare term counts far more than matching "the"
- **Term saturation** — the fifth occurrence of a word adds much less than the second (parameter `k1`)
- **Length normalisation** — long documents don't get to win by containing everything (parameter `b`)

**For a codebase this is not a supplementary technique. It is often the primary one.** When you search for `CreateOrderCommandHandler` you want *that exact symbol*, not the semantically similar `OrderCreationService`. Embeddings blur precisely the distinction you care about. Exception messages, error codes, config keys, NuGet package names — all lexical.

Week 3 has you build both and fuse them. On a code corpus, expect BM25 to carry more weight than most tutorials would suggest.

### 4. Bi-encoders vs cross-encoders

The embedding model above is a **bi-encoder**: it encodes the query and each document *separately*, then compares vectors. That separation is what makes it fast — you embed documents once, at index time, and only the query at query time.

A **cross-encoder** takes query and document *together* and outputs a relevance score. Because it sees both at once with full attention across them, it's substantially more accurate. Because it can't precompute anything, it's far too slow to run over your whole index.

So you use both in a funnel:

```
8,000 chunks → bi-encoder + BM25 → 50 candidates → cross-encoder → 12 chunks → prompt
```

This is called reranking. It is reliably one of the two largest quality wins available.

### 5. The context window is your real constraint

Everything reduces to this. The model can only see what fits in the prompt. Every decision — chunk size, how many chunks to retrieve, how much surrounding context to include — is an allocation decision against a fixed budget.

And it's not just about fitting. Models attend unevenly across long contexts; material in the middle of a large prompt is measurably more likely to be ignored ("lost in the middle"). More retrieved chunks is not monotonically better. There is an optimum, it's usually lower than you'd guess, and you'll find yours empirically in Week 3.

One thing that will bite you specifically: **code tokenises far denser than prose.** English runs about 4 characters per token. C# runs closer to 2.5 — all those braces, `var`, camelCase splits, and namespace qualifiers. A 500-line file is far more tokens than a 500-line document. You'll measure your own ratio on Day 2.

---

## Part 3 — Why code RAG is a different problem

This is the part that isn't in the tutorials. Seven ways your corpus breaks the standard approach.

### 1. Fixed-size chunking destroys code

Splitting every 512 tokens will cut through the middle of a method. Half a method is worse than useless — it embeds poorly, retrieves misleadingly, and gives the model a fragment it can't reason about.

Code has real structural boundaries: namespaces, types, members. **Chunk on the AST, not on token count.** One method per chunk, one property per chunk, with type and namespace attached. That's Days 2–3.

### 2. A chunk without context is unretrievable

Consider this chunk, extracted perfectly along AST lines:

```csharp
public async Task<Guid> Handle(CreateOrderCommand request, CancellationToken ct)
{
    var order = Order.Create(request.CustomerId, request.Items);
    await _repository.AddAsync(order, ct);
    return order.Id;
}
```

Which class? Which project? What's `_repository`? Someone asking "how are orders created in the ordering service" will never retrieve this, because the words "ordering" and "service" appear nowhere in it.

The fix is cheap and it is one of the highest-return moves in the entire plan: **prepend a context header** to the chunk text before embedding it.

```
// File: src/Ordering.Application/Orders/CreateOrderCommandHandler.cs
// Namespace: Ordering.Application.Orders
// Type: CreateOrderCommandHandler : IRequestHandler<CreateOrderCommand, Guid>
// Member: Handle(CreateOrderCommand, CancellationToken)
```

Costs about 40 tokens per chunk. Typically buys 10–20% recall. You build it on Day 3.

### 3. Identifiers need special tokenisation

BM25 tokenises on word boundaries. `CreateOrderCommandHandler` is one token, so a search for "order handler" matches nothing.

The fix is to index both the whole identifier and its subtokens:

```
CreateOrderCommandHandler
  → ["createordercommandhandler", "create", "order", "command", "handler"]
```

This means both an exact symbol search and a natural-language phrase can hit the same chunk. Most code-search implementations that feel broken are broken here. Day 12.

### 4. Generated code will poison your index

A typical C# solution contains a great deal of code no human wrote: `obj/`, `bin/`, `*.Designer.cs`, `*.g.cs`, EF migrations, scaffolded clients. Index it and your retrieval fills with machine-generated noise that is textually similar to everything and semantically relevant to nothing.

Aggressive exclusion on Day 1, before you index anything.

### 5. Your golden set can be objectively scored

This is an advantage, and a real one. For a prose corpus, "was the retrieved context correct?" requires judgement. For code, the ground truth is a **file path and a symbol name**.

"Where is the retry policy configured?" → `src/Infrastructure/Http/ResiliencePolicies.cs`, `AddRetryPolicy`.

That's a string comparison. It means your context recall metric is exact rather than LLM-judged, which makes Week 3's experiments far more trustworthy than they'd be on documents.

### 6. Small chunks mean higher k

Methods are short — often 5 to 30 lines. Retrieving 5 chunks gives the model almost nothing to work with. Where a document system might retrieve 5 chunks of 512 tokens, a code system typically wants **40–60 candidates reranked down to 10–15**. Budget accordingly.

### 7. Relationships matter as much as content

The most useful questions about a codebase are relational: what calls this, what implements this interface, where does this flow end up. Pure similarity search can't answer them — the answer isn't textually similar to the question.

Week 4 adds structural retrieval: a repo map and a symbol graph built at index time from the same AST you're already parsing. This is where a code RAG system stops feeling like search and starts feeling like it understands the codebase.

---

## Part 4 — The stack

| Layer | Choice | Why |
|---|---|---|
| Runtime | WSL2 / Ubuntu, Python 3.12, `uv` | Already set up; the AI ecosystem is Linux-native |
| Parsing | `tree-sitter` + `tree-sitter-c-sharp` | Error-tolerant, fast, same grammar family for every language you'll add later |
| Embeddings | `jinaai/jina-embeddings-v2-base-code` | Trained on code. 768-dim, 8192 context, runs on CPU. Benchmarked against `bge-small` in Week 3 |
| Lexical | `rank-bm25` with identifier-aware tokenisation | Non-optional for code |
| Reranker | `bge-reranker-base` | Local, CPU-viable. Tested against alternatives in Week 3 |
| Vector store | Chroma → Qdrant (Week 4) | Chroma to learn, Qdrant to run |
| Generation | Claude API behind a `Generator` protocol | Swappable to Ollama; on CPU a local 7B will make your sessions painful |
| Eval | RAGAS + exact file-path recall | The second one is your real metric |
| Serving | FastAPI + SSE | Streaming matters more than raw latency |

### The one decision to make yourself

**Which repository.** It should be:

- **One you know well.** You must be able to judge an answer instantly. This outweighs everything else.
- **50k–500k lines.** Smaller and retrieval is trivially easy, so you learn nothing. Larger and your CPU indexing runs get slow enough to break the daily rhythm.
- **Real, not a sample.** Sample projects have no architectural ambiguity, and ambiguity is what makes retrieval hard.
- **Ideally with some documentation** — READMEs, ADRs, XML doc comments. Mixed-modality retrieval is a genuine problem and worth having in scope.

---

## Part 5 — Revised five-week structure

The weeks shift from the document version. Ingestion is now substantially harder, so Week 1 grows; structural retrieval gets its own slot in Week 4.

| Week | Theme | End state |
|---|---|---|
| **1** (Jul 27–31) | Ingestion | AST-chunked, context-headed index of your repo. Working answers by Friday. |
| **2** (Aug 3–7) | Instrumentation & eval | Packaged, traced, 60-case golden set, RAGAS baseline recorded |
| **3** (Aug 10–14) | Retrieval quality | Embedding bake-off, identifier-aware BM25, RRF, reranking. Measured. |
| **4** (Aug 17–21) | Structure & storage | Repo map, symbol graph, Qdrant, incremental indexing |
| **5** (Aug 24–28) | Serving & capstone | FastAPI + SSE, tests, CI, Docker, agentic multi-hop, write-up |

---

## Part 6 — What "hero" means here

By 28 August you should be able to do all of the following without looking anything up:

**Explain**
- Why a bi-encoder and a cross-encoder exist in the same system
- What HNSW is approximating and what it costs you
- Why BM25 still wins on identifier queries in 2026
- What "lost in the middle" means for your k parameter

**Diagnose**
- Read a trace and classify a failure as recall, precision, or faithfulness in under a minute
- Know, from that classification, which lever to reach for

**Build**
- Parse an unfamiliar language with tree-sitter and produce sensible chunks
- Stand up a hybrid retriever with rank fusion from first principles
- Write an eval set that catches regressions rather than flattering the system

**Judge**
- Look at a RAG product and identify what it's doing wrong
- Give a cost, latency and quality estimate for a proposed RAG feature
- Say confidently when RAG is the wrong tool

The last one is the real marker. Plenty of people can build this. Rather fewer can tell you when not to.

---

## Weekend reading

Two papers, in this order. About 90 minutes total.

1. **Lewis et al. 2020**, *Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks* — abstract, sections 1–2, figure 1. Skip the training methodology entirely; you want the inference-time pattern it named.
2. **Liu et al. 2023**, *Lost in the Middle* — the whole thing, it's short. It will change how you think about k.

Then: pick your repository. Clone it. Count the lines. That's Monday's starting point.
