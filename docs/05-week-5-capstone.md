# Week 5 — Serving and Capstone

**24–28 August · Days 21–25**

**Goal:** turn a working system into a deployable one, add agentic multi-hop retrieval, and produce the artifact you'll show people.

---

## Day 21 — Mon 24 Aug · Git-aware incremental indexing

**What you're learning today:** how an index stays current, and why code gives you a re-indexing shortcut that document corpora don't have.

### Hard block (2 hrs)

#### 1. Why this matters now

Re-indexing your repo takes several minutes. That's tolerable for a weekly experiment and intolerable for a system anyone actually uses — code changes hourly. Worse, a stale index is silently wrong: it answers confidently about code that no longer exists.

Code has an advantage documents don't: **git already tracks exactly what changed.**

#### 2. Two layers of change detection

**Layer one — git diff.** Store the indexed commit SHA in collection metadata:

```python
def changed_files(repo: Path, since_sha: str) -> tuple[set[str], set[str]]:
    """Returns (modified_or_added, deleted)."""
    out = subprocess.run(
        ["git", "diff", "--name-status", f"{since_sha}..HEAD"],
        cwd=repo, capture_output=True, text=True, check=True,
    ).stdout
    modified, deleted = set(), set()
    for line in out.splitlines():
        status, _, path = line.partition("\t")
        if status.startswith("D"):
            deleted.add(path)
        elif status.startswith("R"):       # rename: old\tnew
            old, _, new = path.partition("\t")
            deleted.add(old); modified.add(new)
        else:
            modified.add(path)
    return modified, deleted
```

Handle renames explicitly — `git diff` reports them as `R100\told\tnew` and treating that as a single path silently orphans the old file's chunks.

Also handle the uncommitted case: if the working tree is dirty, add `git diff --name-only HEAD` and `git ls-files --others --exclude-standard` to catch modified and untracked files.

**Layer two — content hashes.** Git tells you a file changed; it doesn't tell you whether the *chunks* changed. A whitespace fix or a comment edit changes the file but leaves most chunks byte-identical.

```python
chunk.content_hash = sha256(f"{chunk.header}\n{chunk.body}".encode()).hexdigest()[:16]
```

Re-chunk changed files, compare hashes against stored ones, and embed only genuinely new chunks. Embedding is the expensive step; skipping it is the whole point.

#### 3. The delete path

The half everyone skips, and the one that produces the strangest bugs.

For every deleted or modified file: remove its chunks from the vector store, remove its symbols and edges from the graph, invalidate the repo map, and invalidate any cached answers.

Orphaned chunks are worse than missing ones. A deleted method that still lives in the index will be retrieved and cited, and the citation points at a file that no longer contains it.

#### 4. Rebuild the graph incrementally

Edges are the tricky part. Deleting a file removes its outbound edges cleanly, but *inbound* edges — other files calling into it — still reference symbols that no longer exist. Either:

- Delete edges where either endpoint is gone (simple, correct), or
- Keep them and mark as dangling (more information, more complexity)

Take the simple option. Dangling-edge analysis is a different tool.

#### 5. Wire it up

```bash
coderag index --incremental        # default: diff from stored SHA
coderag index --force              # full rebuild
coderag index --since <sha>        # explicit range
```

Store `indexed_sha`, `indexed_at`, and `index_key` in collection metadata. On query, warn if `HEAD` differs from `indexed_sha` by more than N commits.

### Consolidation (1–1.5 hrs)

1. **Benchmark.** Full rebuild versus incremental after a one-file change. Expect a very large ratio — minutes to a second or two.
2. **Test the delete path.** Delete a file, re-index incrementally, confirm its chunks are gone from the vector store *and* its symbols from the graph. Then ask a question that used to retrieve it.
3. **Test rename.** Rename a file, re-index, confirm no duplicate chunks under both paths.
4. **Consider a git hook.** A `post-commit` hook running `coderag index --incremental` in the background makes the index self-maintaining. Optional, satisfying.

### Reading (30 min)

Vector index update strategies: soft deletes in HNSW, why deletion degrades graph connectivity over time, and when a collection needs a full rebuild. Relevant if this index runs for months.

### ✅ Done when

- Incremental re-index after a one-file change completes in seconds
- Delete and rename paths tested
- Graph updates incrementally alongside the vector store

---

## Day 22 — Tue 25 Aug · Serving, streaming, and caching

**What you're learning today:** where latency actually goes, and why streaming matters more than optimisation.

### Hard block (2 hrs)

#### 1. The API

```bash
uv add fastapi uvicorn sse-starlette diskcache
```

```
POST /ask          → Answer (JSON)
GET  /ask/stream   → SSE
GET  /symbols/{name}/callers
GET  /symbols/{name}/implementations
GET  /health       → index_key, indexed_sha, chunk count, staleness
GET  /stats
```

The symbol endpoints are worth exposing directly. They're deterministic graph queries with no LLM in the path — instant, free, and genuinely useful on their own.

#### 2. Two CPU-bound stages, not one

This is the trap. Your pipeline has two stages that saturate a CPU core:

- **Query embedding** — ~20–50ms
- **Reranking 50 pairs** — ~500–2000ms

Both block the event loop if called directly from an async handler. Under concurrent load your API will appear to hang, and the reranker is the bigger offender by an order of magnitude.

```python
executor = ThreadPoolExecutor(max_workers=2)

async def retrieve_async(query: str) -> list[ScoredChunk]:
    loop = asyncio.get_running_loop()
    qvec = await loop.run_in_executor(executor, embedder.embed_query, query)
    ...
    return await loop.run_in_executor(executor, reranker.rerank, query, candidates, keep)
```

`sentence-transformers` releases the GIL during the actual torch forward pass, so threads genuinely help here. Keep `max_workers` low — 2 to 4 — because these are compute-bound and oversubscribing just adds contention.

Verify with a concurrency test: fire 10 simultaneous requests and confirm `/health` still responds immediately.

#### 3. Streaming

Generation dominates your latency budget — typically 1.5 to 3 seconds against a few hundred milliseconds for everything else. You cannot make it much faster, but you can make it feel faster by an order of magnitude.

Event sequence:

```
event: retrieved     data: {"count": 12, "files": [...]}    ← immediately after retrieval
event: token         data: {"text": "The order total is "}
event: token         data: {"text": "calculated in "}
...
event: citations     data: [{"index": 1, "file_path": "...", ...}]
event: done          data: {"latency_ms": {...}, "trace_id": "..."}
```

Sending `retrieved` before generation starts is a small touch worth a lot — the user sees which files were found within ~400ms, which is enough to know whether the answer is going to be relevant.

#### 4. Caching

Two caches, different keys:

```python
# Query embedding — invalidated by model change only
emb_key = f"emb:{settings.embedding_model}:{sha256(query)}"

# Full answer — invalidated by anything that changes the answer
ans_key = f"ans:{settings.index_key}:{indexed_sha}:{prompt_version}:{sha256(normalised_query)}"
```

**Cache invalidation is the classic bug here, and the consequence is worse than usual.** A stale answer cache during a Week 3–style sweep means you compare configuration A against cached results from configuration A, conclude the change did nothing, and revert something that worked. Including `index_key` and `indexed_sha` in the key prevents it structurally.

Add `--no-cache` to the CLI and bypass the cache entirely during eval runs. Belt and braces.

#### 5. Minimal UI

One `static/index.html`. Textarea, streaming answer, file list with line ranges. No framework, no build step. It exists so you can demo the thing without a terminal.

#### 6. Benchmark

`scripts/bench.py` — 50 queries, cold and warm cache, p50/p95/p99, per-stage breakdown.

Target: p95 under 3s cold. Then profile, and confirm generation dominates before optimising anything else. If retrieval exceeds 20% of the budget, the reranker is the place to look — smaller model, fewer candidates, or ONNX export.

### Consolidation (1–1.5 hrs)

1. **Concurrency test.** 10 simultaneous requests; confirm the event loop stays responsive and latency degrades gracefully rather than collapsing.
2. **Cache correctness test.** Query, re-index, query again — the answer must be recomputed, not served stale.
3. **Time to first token.** Measure it separately from total latency. It's the number that determines whether the system feels fast, and it should be under 500ms.

### Reading (30 min)

Perceived performance and streaming interfaces; why time-to-first-token dominates user-perceived latency and why optimising total time is often the wrong target.

### ✅ Done when

- p95 under 3s, time to first token under 500ms
- Event loop responsive under 10 concurrent requests
- Cache invalidates correctly on re-index
- Web UI streams

---

## Day 23 — Wed 26 Aug · Tests, CI, and containers

**What you're learning today:** how to test a system whose output is non-deterministic by asserting only on the parts that aren't.

### Hard block (2 hrs)

#### 1. The testing principle

You cannot assert `answer == expected`. You *can* assert on everything upstream of the model, which is most of the system.

| Layer | Deterministic? | Test approach |
|---|---|---|
| Chunking | Yes | Exact assertions |
| Tokenisation | Yes | Exact assertions |
| Graph extraction | Yes | Exact assertions |
| RRF | Yes | Hand-worked example |
| Vector retrieval | Yes, given a fixed index | Fake embedder |
| Reranking | Yes | Fake scores |
| Generation | No | Don't unit test |
| End to end | No | Golden-case retrieval assertions |

#### 2. Fakes

```python
class FakeEmbedder:
    dimension = 8
    model_name = "fake"
    def _vec(self, text: str) -> np.ndarray:
        h = hashlib.sha256(text.encode()).digest()[:8]
        v = np.frombuffer(h, dtype=np.uint8).astype(np.float32) - 128
        return v / np.linalg.norm(v)
    def embed_query(self, text): return self._vec(text)
    def embed_documents(self, texts): return np.stack([self._vec(t) for t in texts])
```

Deterministic, instant, no model download, no network. Your whole suite should run offline in under a minute — otherwise you won't run it.

#### 3. The tests that matter

**Chunking** — coverage (every byte of a fixture file appears in some chunk or is deliberately dropped), header correctness, file-scoped namespace handling, oversized method splitting, small-member merging.

Keep a `tests/fixtures/` directory with real C# files exercising each case: a file-scoped namespace, a nested class, a record, an interface with default implementations, a 400-line method, a class of auto-properties.

**Tokenisation** — the table from Day 13, exactly.

**RRF** — your hand-worked example, asserting exact fused order.

**Graph extraction** — a fixture file with known calls and implementations; assert the exact edge set.

**Citation parsing** — every malformed form.

**Golden regression** — 15 cases from the golden set, asserting `hit@10` on file paths. No LLM, so it's deterministic and fast. This is your merge gate.

#### 4. CI

```yaml
- uv sync
- uv run ruff check .
- uv run ruff format --check .
- uv run mypy --strict coderag/
- uv run pytest -q
```

The golden regression needs an index. Build a small one from `tests/fixtures/` in CI rather than checking in a real index — fast, hermetic, and it doesn't leak your codebase into the repo.

Gate policy in `CONTRIBUTING.md`: block on golden-case failures, flag but don't block retrieval metric regressions under 2%.

#### 5. Containers

Multi-stage `Dockerfile`. The key detail: **cache model weights in their own layer**, or every build re-downloads several hundred megabytes.

```dockerfile
FROM python:3.12-slim AS models
RUN pip install --no-cache-dir sentence-transformers
RUN python -c "\
from sentence_transformers import SentenceTransformer, CrossEncoder; \
SentenceTransformer('jinaai/jina-embeddings-v2-base-code', trust_remote_code=True); \
CrossEncoder('BAAI/bge-reranker-base')"

FROM python:3.12-slim
COPY --from=models /root/.cache/huggingface /root/.cache/huggingface
...
```

`docker-compose.yml` with api and qdrant, healthchecks, and `depends_on: condition: service_healthy`. Mount the repo read-only.

Then: clone fresh into `/tmp`, `docker compose up`, and fix everything you had to do by hand.

### Consolidation (1–1.5 hrs)

1. **Time the clean-clone path**, write it into the README as a quickstart with the real number.
2. **Break something deliberately** — change a chunk boundary — and confirm the golden regression catches it.
3. **Check image size.** Over 3GB means the model layer isn't shared properly.

### Reading (30 min)

Testing non-deterministic systems: property-based testing, metamorphic testing (assert relationships between outputs rather than outputs themselves), and snapshot testing with tolerance.

### ✅ Done when

- Full suite runs offline in under 60 seconds
- CI green
- Fresh clone → one command → working system
- Regression suite catches a deliberate break

---

## Day 24 — Thu 27 Aug · Agentic retrieval

**What you're learning today:** when letting the model drive retrieval beats a fixed pipeline, and what it costs.

### Hard block (2 hrs)

#### 1. What's still failing

Even after Week 4, some questions need *sequential* retrieval — where the second search depends on what the first one found.

> "When an order is confirmed, which external services get notified?"

Find `Order.Confirm()`. See it raises `OrderConfirmedEvent`. Now search for handlers of that event. See one calls `INotificationService`. Now find its implementation. Three dependent searches; no single query finds the answer, and graph expansion only goes one hop.

#### 2. The graph as tools

This is where Day 18's work pays off unexpectedly. Instead of one `search` tool, expose the graph:

```python
tools = [
    {"name": "search_code",
     "description": "Semantic and keyword search over the codebase.",
     "input_schema": {"query": "str", "project": "str?", "include_tests": "bool?"}},
    {"name": "get_symbol",
     "description": "Fetch the full source of a named type or member.",
     "input_schema": {"name": "str"}},
    {"name": "get_callers",
     "description": "Find all call sites of a method.",
     "input_schema": {"symbol": "str"}},
    {"name": "get_implementations",
     "description": "Find classes implementing an interface.",
     "input_schema": {"interface": "str"}},
]
```

These are precise, deterministic, and cheap. The model can navigate the codebase the way you would in an IDE — search, then jump to definition, then find usages — rather than making everything a similarity query.

#### 3. The loop

```python
async def agentic_answer(question: str, max_iterations: int = 5) -> Answer:
    messages = [{"role": "user", "content": question}]
    for _ in range(max_iterations):
        resp = await generator.generate_with_tools(SYSTEM, messages, tools)
        if resp.stop_reason != "tool_use":
            break
        results = [await dispatch(call) for call in resp.tool_calls]
        messages.append({"role": "assistant", "content": resp.content})
        messages.append({"role": "user", "content": results})
    return build_answer(resp, accumulated_context)
```

Hard-cap iterations. Without it, a model that can't find something will search forever, and each iteration is a full generation call.

Track cumulative token spend per query and abort past a threshold.

#### 4. Measure honestly

Run on the multi-hop subset and the full set **separately**. The aggregate will hide what's happening.

Expected shape:

| Metric | Fixed pipeline | Agentic |
|---|---|---|
| Multi-hop accuracy | 0.35 | 0.75 |
| Overall accuracy | 0.82 | 0.85 |
| p95 latency | 2.8s | 11s |
| Tokens per query | 4,500 | 22,000 |

Substantially better on hard questions, marginally better overall, dramatically more expensive. **That's not a disappointing result — it's the correct result**, and understanding it is the point.

#### 5. Route rather than replace

Put it behind a flag, then route:

- Simple lookups → fixed pipeline
- Questions with multi-hop markers ("when X happens, what...", "trace", "end to end", "which services") → agentic
- Fixed pipeline returning low confidence → escalate to agentic

The escalation route is the elegant one: try fast, fall back to thorough.

### Consolidation (1–1.5 hrs)

1. **Read three agentic traces in full.** What did the model search for? Did it use `get_callers` when it should have? Bad tool descriptions show up here as odd tool choices — the description *is* the prompt.
2. **Find a runaway.** Some question will hit the iteration cap. Understand why. Usually the answer genuinely isn't in the repo and the model won't give up — which means your abstention instruction needs to apply to the agentic path too.
3. **Cost per query.** Multiply by an imagined 1,000 queries/day. This is the kind of number that makes an architecture argument concrete.

### Reading (30 min)

Self-RAG and ReAct-style tool-use loops. Note the recurring finding: giving models *precise* tools (a graph query) beats giving them *more* of a fuzzy tool (more similarity search).

### ✅ Done when

- Multi-hop subset substantially improved
- Cost and latency quantified against the fixed pipeline
- Routing implemented, flag-gated
- Three traces read and understood

---

## Day 25 — Fri 28 Aug · Final evaluation and capstone

### Hard block (2 hrs)

#### 1. The full run

`make eval` on all 60 cases, both fixed and agentic paths. Save as the final run.

#### 2. The table

This is the most persuasive artifact you'll produce. Every intervention, in order, with its measured delta:

| Day | Change | recall@10 | Δ | Faithfulness | Δ |
|---|---|---|---|---|---|
| 10 | Baseline (vector only) | 0.61 | — | 0.64 | — |
| 11 | Chunking ablations | 0.67 | +0.06 | 0.66 | +0.02 |
| 12 | Code embedding model | 0.72 | +0.05 | 0.69 | +0.03 |
| 13–14 | Hybrid BM25 + RRF | 0.84 | +0.12 | 0.78 | +0.09 |
| 15 | Cross-encoder rerank | 0.87 | +0.03 | 0.84 | +0.06 |
| 16 | Prompt hardening | 0.87 | 0.00 | 0.91 | +0.07 |
| 17 | Repo map | 0.88 | +0.01 | 0.92 | +0.01 |
| 18–19 | Symbol graph expansion | 0.93 | +0.05 | 0.94 | +0.02 |
| 24 | Agentic (multi-hop only) | — | — | — | — |

Plus the per-kind table, before and after. The `cross_cutting` and `trace` rows are the story.

#### 3. README

- What it is, in two sentences
- Quickstart with the real clean-clone timing
- Architecture diagram
- Results table
- Design decisions with their justifying measurement — this section is what makes it a portfolio piece rather than a tutorial repo
- Known limitations, stated plainly: syntactic-only resolution, single-language, CPU-bound reranking
- What's next

#### 4. Architecture diagram

Indexing path and query path. Show the graph as a parallel store to the vector index, and show which stages are CPU-bound.

### Consolidation (1–1.5 hrs)

**Write-up #5, the capstone.** The whole build with numbers. Structure that works:

1. What I built and why
2. The unflattering baseline
3. The four changes that mattered, with deltas
4. The thing nobody builds (symbol graph) and why it works for code specifically
5. What I'd do differently
6. What's still wrong

Then the retro, in `NOTES.md`:

- Which of your Day 5 failure-class predictions were right?
- Which intervention over-delivered? Which was theatre?
- What took three times longer than expected?
- What would you build first if starting over?

### Reading (30 min)

Pick the next domain, deliberately:

- **Evals infrastructure** — the most transferable skill; every serious AI team needs it and few do it well
- **Fine-tuning** — embedding models fine-tuned on your own code/query pairs
- **Agentic coding tools** — the natural extension; you have retrieval, add editing
- **Multi-language** — the same pipeline with TypeScript and SQL grammars

### ✅ Done when

- Final eval run recorded
- Before/after table complete
- README publishable
- Capstone write-up published
- Retro written

---

## Where you land

Five weeks, roughly 90 hours, and you can:

**Explain** — why bi-encoders and cross-encoders coexist; what HNSW approximates; why BM25 still wins on identifiers; why AST chunking beats fixed-size for code; when a symbol graph beats similarity search.

**Diagnose** — read a trace and classify a failure in under a minute, and know which lever it implies.

**Build** — parse an unfamiliar language with tree-sitter; hybrid retrieval with rank fusion from first principles; an eval set that catches regressions.

**Judge** — cost, latency and quality estimates for a proposed RAG feature; and when RAG is the wrong tool.

That last one is the real marker. Plenty of people can build this. Rather fewer can tell you when not to.

---

## What's genuinely still missing

Worth stating in the README, because knowing the gaps is part of the competence:

- **Semantic resolution.** Name-based edges are approximate. Roslyn would fix this and would be the right call for a production tool.
- **Single language.** The pipeline generalises; you've only proven it on C#.
- **No conversational memory.** Every question is independent. Follow-ups ("what about the async version?") don't work.
- **CPU-bound reranking** caps throughput at roughly one query per second per core.
- **No access control.** Every user sees everything. Real deployments need per-repo and per-branch scoping.
- **Eval set is yours alone.** Sixty cases written by one person reflect one person's idea of what matters.
