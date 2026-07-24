# Week 3 — Retrieval Quality

**10–14 August · Days 11–15**

**Goal:** every retrieval decision from Week 1 tested against a number, plus hybrid search and reranking. Expect the largest quality gains of the whole plan this week.

**The discipline:** every hard block ends with `make retrieval-eval`. Keep changes that improve recall@10 by more than 0.02; revert everything else. Never keep a change that didn't move a number, however sensible it seemed.

---

## Day 11 — Mon 10 Aug · Testing the chunking decisions

**What you're learning today:** that plausible design decisions are hypotheses, and most of yours are about to be tested for the first time.

Week 1 made four claims. Today you find out whether they're true on *your* repo.

### Hard block (2 hrs)

#### 1. Set up the ablation runner

Because `index_key` derives from the index-affecting config, each variant writes to its own collection automatically. Build a runner that takes a list of config overrides, re-indexes, runs the fast eval, and collects results:

```python
VARIANTS = [
    ("baseline",          {}),
    ("no_header",         {"include_header": False}),
    ("no_doc_comments",   {"include_doc_comments": False}),
    ("no_type_summaries", {"include_type_summaries": False}),
    ("merge_0",           {"merge_below_tokens": 0}),
    ("merge_50",          {"merge_below_tokens": 50}),
    ("max_300",           {"max_chunk_tokens": 300}),
    ("max_600",           {"max_chunk_tokens": 600}),
]
```

Eight full re-indexes. On CPU at 8,000 chunks that's roughly 30–60 minutes of wall clock — start it running and use the time to write the analysis script.

#### 2. The four claims under test

**Claim 1: context headers are worth 10–20% recall.** This is the big one, and the one I asserted most confidently in Week 1. `no_header` vs `baseline` on recall@10 settles it.

If the gain is smaller than claimed, look at *which* question kinds moved. Headers should help `behaviour` and `usage` questions most (natural-language phrasing matching namespace and type words) and `symbol` questions least (the symbol name is in the body anyway).

**Claim 2: doc comments are the most retrievable text on a symbol.** Depends heavily on your repo. A codebase with thorough XML docs will show a large drop when you remove them; one with `/// <summary>Gets or sets the Id.</summary>` everywhere will show noise. Either result is informative — and if the second, consider filtering out doc comments below a usefulness threshold rather than dropping them entirely.

**Claim 3: type summaries answer a different question than method chunks.** Look at the per-kind table. Type summaries should carry `behaviour` and `cross_cutting`; method chunks should carry `symbol`. If removing type summaries barely moves anything, they're costing you index size for nothing.

**Claim 4: merging tiny members improves precision.** Compare `merge_0`, baseline (25), and `merge_50`. Watch MRR in particular — merging should push trivial property chunks out of the top ranks. Also record chunk counts; the merge threshold is your main lever on index size.

#### 3. Analysis

For each variant record: recall@10, MRR, per-kind breakdown, chunk count, index build time. Plot recall@10 against chunk count — you're looking for the point where more chunks stop buying quality.

### Consolidation (1–1.5 hrs)

1. **Read the cases each variant broke.** `make compare` gives you named regressions. Open three of them and look at what was retrieved instead. This is where the real understanding is: not "headers help by 0.14" but "without headers, questions phrased around the domain concept fail because the domain words only appear in the namespace."
2. **Lock in the winning configuration** and re-index.
3. **Write the results into `NOTES.md` as a table.** Day 25's write-up is largely built from these tables, and reconstructing them later is impossible.

### Reading (30 min)

*Lost in the Middle* again if you skipped it, otherwise the literature on chunk granularity for retrieval. The recurring finding across corpora: retrieval quality peaks at a moderate chunk size and degrades in both directions, and the optimum depends on how the queries are phrased more than on the content.

### ✅ Done when

- Eight variants measured, results tabled
- Winning config locked in and re-indexed
- You can state, with a number, whether context headers were worth building

---

## Day 12 — Tue 11 Aug · The embedding bake-off

**What you're learning today:** how much the choice of embedding model actually matters, and how to evaluate one on your own data rather than on a leaderboard.

### Hard block (2 hrs)

#### 1. The contenders

| Model | Dim | Notes |
|---|---|---|
| `jinaai/jina-embeddings-v2-base-code` | 768 | Current default. Code-trained, 8192 context. |
| `BAAI/bge-small-en-v1.5` | 384 | Prose-trained. The control. Needs a query prefix. |
| `nomic-ai/nomic-embed-text-v1.5` | 768 | Strong general model, prefix-based task routing. |
| `voyage-code-3` (API) | 1024 | Optional. Code-specialised, generally state of the art. Costs money. |

Include `bge-small` specifically as a control. The primer asserted that prose-trained models underperform on code; this is where you verify it rather than believe it.

#### 2. Handle the prefix problem properly

This is where the Day 6 protocol design earns its keep. Each model has different query/document conventions:

```python
class BGEEmbedder:
    QUERY_PREFIX = "Represent this sentence for searching relevant passages: "
    def embed_query(self, text: str) -> np.ndarray:
        return self._encode([self.QUERY_PREFIX + text])[0]
    def embed_documents(self, texts: list[str]) -> np.ndarray:
        return self._encode(texts)          # no prefix

class NomicEmbedder:
    def embed_query(self, text: str) -> np.ndarray:
        return self._encode(["search_query: " + text])[0]
    def embed_documents(self, texts: list[str]) -> np.ndarray:
        return self._encode(["search_document: " + t for t in texts])

class JinaCodeEmbedder:
    # no prefixes
    ...
```

**Getting a prefix wrong is a silent 5–15% recall loss.** Nothing errors. The scores just come out slightly worse and you conclude the model is mediocre. Read each model card specifically for this, and sanity-check by embedding the same text as query and document and confirming the vectors differ where they should.

#### 3. Run the bake-off

Each model needs its own collection (different dimensions anyway — the `index_key` handles it). Record per model:

- recall@10, MRR, per-kind breakdown
- index build time
- query latency (p50 embedding time)
- index size on disk

#### 4. Read the per-kind table, not the average

The interesting result is usually not "model X wins." It's that models differ by question kind. A code-trained model typically wins on `symbol` and `usage` questions; a strong general model can win on `behaviour` questions, which are phrased in natural language about intent rather than in code vocabulary.

If two models are close in aggregate but win different kinds, note it. That's a genuine argument for an ensemble, though not one you should act on yet — hybrid retrieval on Day 13 addresses the same weakness more cheaply.

### Consolidation (1–1.5 hrs)

1. **Cost-adjusted decision.** If `voyage-code-3` wins by 0.04 but costs money per query and adds network latency to every request, is it worth it? Write the reasoning down. This is the kind of judgement the "hero" definition in the primer was pointing at.
2. **Inspect a disagreement.** Find a case one model gets and another misses. Look at the chunk. Usually the difference is vocabulary: one model has learned that `IRequestHandler` relates to "command processing," the other hasn't.
3. Lock in the winner, re-index, commit the results table.

### Reading (30 min)

The MTEB and CoIR benchmark methodology. Specifically: what CodeSearchNet's query distribution looks like (docstrings paired to functions) and why that makes it an imperfect proxy for "developer asks a question in English." This is the argument for trusting your own 60 cases over a leaderboard.

### ✅ Done when

- Four models measured on identical cases
- Prefix conventions verified per model
- Winner chosen with quality, latency, and cost reasoning recorded

---

## Day 13 — Wed 12 Aug · Identifier-aware lexical search

**What you're learning today:** why BM25 remains essential in 2026, and why the standard tokeniser is wrong for code.

This is the most code-specific day in the plan.

### Hard block (2 hrs)

#### 1. Why this matters more than the tutorials suggest

Someone searching for `CreateOrderCommandHandler` wants that exact type. An embedding model has learned that `CreateOrderCommandHandler`, `OrderCreationService`, and `PlaceOrderHandler` are all semantically similar — which is exactly the distinction being asked about. Embeddings blur; BM25 doesn't.

The same applies to exception type names, error codes, config keys, NuGet package names, route templates, and connection string names. A large fraction of real developer queries are lexical.

#### 2. The tokeniser

Off-the-shelf tokenisation makes `CreateOrderCommandHandler` a single token, so "order handler" matches nothing. The fix is to index the whole identifier *and* its subtokens:

```python
import re

_CAMEL = re.compile(
    r"[^A-Za-z0-9]+"              # separators
    r"|(?<=[a-z0-9])(?=[A-Z])"    # fooBar    → foo | Bar
    r"|(?<=[A-Z])(?=[A-Z][a-z])"  # HTTPResp  → HTTP | Resp
)

def code_tokens(text: str) -> list[str]:
    out: list[str] = []
    for raw in re.split(r"[^A-Za-z0-9_]+", text):
        raw = raw.strip("_")
        if not raw:
            continue
        out.append(raw.lower())                  # whole identifier
        parts = [p for p in _CAMEL.split(raw) if p]
        if len(parts) > 1:
            out.extend(p.lower() for p in parts) # subtokens
    return out
```

Verify against these, they're your unit tests:

| Input | Expected tokens |
|---|---|
| `CreateOrderCommandHandler` | `createordercommandhandler, create, order, command, handler` |
| `HTTPResponseCode` | `httpresponsecode, http, response, code` |
| `_orderRepository` | `orderrepository, order, repository` |
| `snake_case_name` | `snake, case, name` |
| `IOrderRepository` | `iorderrepository, i, order, repository` |

The `IOrderRepository` case is worth thinking about. The leading `I` splits off as a useless single-character token. Either drop single characters or, better, special-case the C# interface convention so `IOrderRepository` also emits `orderrepository` — which makes a search for "order repository" match both the interface and its implementations.

Indexing both the whole form and the parts means an exact symbol search still ranks the exact match highest (it matches on the rare full token, which has enormous IDF) while a natural-language phrase can also find it.

#### 3. Build the retriever

```bash
uv add rank-bm25
```

`coderag/lexical.py` — a `BM25Retriever` satisfying the `Retriever` protocol. Tokenise `header + body` for each chunk, build the index, persist it (pickle is fine; it rebuilds in seconds anyway).

On stopwords: C# keywords like `public`, `void`, `var`, `return` appear in nearly every chunk, so their IDF is near zero and BM25 already discounts them. An explicit stoplist adds little. Don't bother unless you measure a gain.

#### 4. Measure BM25 alone

Run the fast eval with BM25 as the only retriever. **Look at the per-kind table.** The expected pattern:

- `symbol` — BM25 wins clearly, often by a lot
- `config` — BM25 wins (config keys are literal strings)
- `behaviour` — vector wins (the question shares no vocabulary with the code)
- `usage` — mixed
- `cross_cutting` — BM25 may win on interface names specifically

### Consolidation (1–1.5 hrs)

1. **Build the win/loss table.** For all 60 cases, mark whether BM25 or vector retrieval ranked the correct file higher. Then characterise the pattern in prose in `NOTES.md`. This is the argument that justifies tomorrow's work, and articulating it clearly is the point of the exercise.
2. **Unit-test the tokeniser** against the table above.
3. **Find a case where BM25 fails badly** and understand why. Usually: the question uses domain vocabulary that appears nowhere in the code, because the code names things after patterns rather than after the business concept.

### Reading (30 min)

Robertson & Zaragoza, *The Probabilistic Relevance Framework: BM25 and Beyond* — sections on term saturation (`k1`) and length normalisation (`b`). Then consider whether your default `b=0.75` is right for code, where chunk length varies far more than in a document corpus. Worth a quick sweep tomorrow if the tuning is cheap.

### ✅ Done when

- Tokeniser passes all cases in the table
- BM25-only scores recorded with per-kind breakdown
- Win/loss pattern characterised in writing

---

## Day 14 — Thu 13 Aug · Hybrid retrieval with rank fusion

**What you're learning today:** how to combine retrievers whose scores aren't comparable, implemented from the paper.

### Hard block (2 hrs)

#### 1. Why not just add the scores

BM25 returns unbounded scores that depend on corpus statistics — maybe 4.2, maybe 31.7. Cosine similarity returns values in a narrow band, often 0.6 to 0.9. Adding or averaging them means the BM25 score dominates arbitrarily, and normalising per query is unstable because the score distributions shift with each query.

Reciprocal Rank Fusion sidesteps this entirely by discarding the scores and using only the **ranks**:

$$\text{RRF}(d) = \sum_{i} \frac{1}{k + \text{rank}_i(d)}$$

Rank 1 contributes `1/61` with `k=60`, rank 2 contributes `1/62`, and so on. The `k` constant flattens the curve — a large `k` means the difference between rank 1 and rank 10 matters less, so a document ranked moderately by both retrievers can beat one ranked first by only one.

#### 2. Implement it from the paper

Ten lines. **Write it yourself**; don't import it. This is one of the two places in the plan where people ship subtly broken code that still produces plausible output.

```python
def rrf(
    rankings: list[list[str]],          # each list: chunk_ids in rank order
    k: int = 60,
    weights: list[float] | None = None,
) -> dict[str, float]:
    weights = weights or [1.0] * len(rankings)
    scores: dict[str, float] = defaultdict(float)
    for ranking, w in zip(rankings, weights, strict=True):
        for rank, chunk_id in enumerate(ranking, start=1):
            scores[chunk_id] += w / (k + rank)
    return scores
```

The classic bug is enumerating from 0, which makes rank-0 contribute `1/k` and rank-1 contribute `1/(k+1)` — a much larger gap at the top than intended. It still works, just worse, and you'd never notice without measuring. **Verify on paper first:** two rankings, five documents each, compute the fused order by hand, assert it in a test.

#### 3. Retrieve wide, then fuse

```python
vector_hits = vector_retriever.retrieve(query, k=50)
bm25_hits   = bm25_retriever.retrieve(query, k=50)
fused = rrf([ids(vector_hits), ids(bm25_hits)], k=settings.rrf_k,
            weights=[1 - settings.bm25_weight, settings.bm25_weight])
```

Retrieve 50 from each. Fusion can only reorder what it's given, so a narrow candidate set caps your ceiling.

#### 4. Sweep

Two parameters:

- `rrf_k` ∈ {10, 20, 60, 100}
- `bm25_weight` ∈ {0.3, 0.4, 0.5, 0.6, 0.7}

Twenty combinations, but no re-indexing required — both indexes already exist, so this is pure query-time work and runs in minutes.

Given Day 13's finding that lexical matters heavily for code, expect the optimal `bm25_weight` to sit at or above 0.5. If it lands much higher, that's a real result worth writing up.

### Consolidation (1–1.5 hrs)

1. **Check the cases BM25 won on Day 13.** Are they fixed in the hybrid? If not, your fusion has a bug or the weight is wrong.
2. **Check for regressions.** Fusion occasionally hurts: a case where the vector retriever ranked the answer first can drop if BM25 ranked something else first and the weighting favours it. Read those cases; they tell you whether your weight is over-tuned to one question kind.
3. **Per-kind sanity.** Hybrid should improve or match every kind. A kind that got worse means the weight is wrong for that kind — which is a real argument for query-dependent routing, a good Week 5 stretch goal but not something to build now.

### Reading (30 min)

Cormack, Clarke & Büttcher on RRF. Note how simple the method is relative to the learned fusion approaches it outperformed, and why rank-based combination is robust to exactly the score-scale problem that breaks naive blending.

### ✅ Done when

- RRF verified against a hand-worked example in a test
- Hybrid beats both single retrievers on recall@10
- `rrf_k` and `bm25_weight` chosen from the sweep
- No question kind regressed

Hybrid retrieval is usually the single largest win in the entire plan. Expect a substantial jump.

---

## Day 15 — Fri 14 Aug · Reranking and the funnel

**What you're learning today:** the precision/latency tradeoff, and where the first real latency cost enters the system.

### Hard block (2 hrs)

#### 1. Cross-encoders, concretely

Your bi-encoder embedded query and chunk separately, so their vectors were computed without reference to each other. A cross-encoder concatenates them and runs full attention across both, producing a single relevance score. It sees which specific tokens in the query align with which tokens in the code.

That's why it's more accurate, and why it can't be precomputed — the score depends on the pair, so there are `queries × chunks` of them.

```python
from sentence_transformers import CrossEncoder
reranker = CrossEncoder("BAAI/bge-reranker-base", max_length=512)
scores = reranker.predict([(query, c.embed_text) for c in candidates])
```

`max_length=512` truncates. Some of your chunks exceed it, and truncation cuts the *end* — which for a method is the return statement, often the most informative part. Consider passing `header + first N tokens + last N tokens` for oversized chunks, and measure whether it helps.

#### 2. Try two rerankers

| Model | Notes |
|---|---|
| `BAAI/bge-reranker-base` | Strong general reranker, mostly prose-trained |
| `jinaai/jina-reranker-v2-base-multilingual` | Handles code explicitly, longer context |

Same argument as Day 12: don't assume the code-specific one wins, measure it.

#### 3. Tune the funnel

The funnel has two numbers: how many candidates to rerank, and how many to keep.

| retrieve | keep | What it tests |
|---|---|---|
| 20 | 8 | Cheap |
| 50 | 12 | Default |
| 50 | 20 | Does more context help generation? |
| 100 | 12 | Does a wider net find more? |

**Record latency for every configuration.** This is the first change that costs real time — a cross-encoder scoring 50 pairs on CPU takes roughly 0.5–2 seconds depending on chunk length and cores. That may be a third of your latency budget.

Two metrics diverge here and you need both:

- **recall@keep** — did reranking retain the right file in the final set?
- **MRR** — did it move the right file *up*?

Reranking should improve MRR sharply while leaving recall roughly flat or slightly down (it can only lose files, never add them). If recall drops meaningfully, your candidate set is too narrow or the reranker is mis-scoring code.

#### 4. Context ordering

While you're here, test the lost-in-the-middle claim from the primer. Three orderings of the final chunks in the prompt:

- descending by score
- ascending by score
- **best-first, second-best-last, weakest in the middle**

This needs the LLM-judged metrics rather than retrieval metrics — it changes nothing about what's retrieved, only how it's presented. Run `make eval` for each. It's a two-line change and typically worth a small but real gain.

### Consolidation (1–1.5 hrs)

**Write-up #3:** *"Hybrid search and reranking on a C# codebase: what actually moved the numbers."*

Publish the per-kind before/after table across the whole week. The `bm25_weight` result is the genuinely interesting finding — most RAG writing treats lexical search as a minor supplement, and if your data shows it carrying half the weight or more on a code corpus, that's worth stating plainly.

Then: run `make eval` in full and compare against the Day 10 baseline. Write the week's cumulative delta into `NOTES.md`.

### Reading (30 min)

Bi-encoder versus cross-encoder architectures; the ColBERT late-interaction approach as the middle ground (per-token embeddings with cheap MaxSim scoring). Worth knowing about as the answer to "can I get cross-encoder quality at bi-encoder speed" — partially, at significant index-size cost.

### ✅ Done when

- Two rerankers compared
- Funnel chosen with a quality/latency table
- Context ordering tested with LLM-judged metrics
- Full `make eval` run and compared to the Day 10 baseline
- Write-up published

---

## Week 3 retrospective

You should be able to explain, without notes:

- Why context headers help, backed by your own ablation number
- Why prose-trained embedding models underperform on code, and by how much on your repo
- Why `CreateOrderCommandHandler` must be indexed as five tokens, not one
- Why RRF uses ranks rather than scores, and what breaks if you blend scores
- Why a cross-encoder can't replace the bi-encoder, only follow it

**What you've built:** hybrid retrieval with tuned rank fusion, reranking, and every chunking decision validated against measurements. `symbol`, `behaviour`, `config`, and `usage` questions should be in decent shape.

**What's still broken, deliberately:** `cross_cutting` and `trace` questions. "Which classes implement `IOrderRepository`?" and "what calls `SendConfirmationEmail`?" are still poor, and no amount of similarity tuning will fix them — the answer isn't textually similar to the question. That's a structural problem requiring a structural solution.

Week 4 builds it.
