# Week 2 — Instrumentation and Evaluation

**3–7 August · Days 6–10**

**Goal:** by Friday you have a properly structured package, full request tracing, a 60-case golden set, and a recorded baseline. Nothing gets tuned this week.

This is the week that separates engineering from guessing. Week 1 built something that works. Week 2 builds the instruments that let you tell whether a change made it better — and for a code corpus you get a much sharper instrument than document RAG allows.

---

## Day 6 — Mon 3 Aug · Package structure and configuration

**What you're learning today:** why every knob has to be config-driven before you start experimenting, and how Python's structural typing differs from the interfaces you know.

### Hard block (2 hrs)

#### 1. Protocols

`coderag/protocols.py`:

```python
from typing import Protocol, runtime_checkable
import numpy as np

@runtime_checkable
class Embedder(Protocol):
    dimension: int
    model_name: str
    def embed_documents(self, texts: list[str]) -> np.ndarray: ...
    def embed_query(self, text: str) -> np.ndarray: ...

@runtime_checkable
class VectorStore(Protocol):
    def upsert(self, chunks: list[Chunk], vectors: np.ndarray) -> None: ...
    def search(self, vector: np.ndarray, k: int,
               where: dict[str, Any] | None = None) -> list[ScoredChunk]: ...
    def count(self) -> int: ...
    def reset(self) -> None: ...

@runtime_checkable
class Retriever(Protocol):
    def retrieve(self, query: str, k: int) -> list[ScoredChunk]: ...

@runtime_checkable
class Reranker(Protocol):
    def rerank(self, query: str, chunks: list[ScoredChunk],
               keep: int) -> list[ScoredChunk]: ...

@runtime_checkable
class Generator(Protocol):
    def generate(self, system: str, user: str) -> GenerationResult: ...
```

`Retriever` is the one that pays off soonest. On Day 13 you write a BM25 retriever, on Day 14 a fusion retriever that wraps two others — and because they all satisfy the same protocol, the pipeline never changes.

The C# adjustment worth internalising: nothing declares `: IRetriever`. A class satisfies `Retriever` by having a matching `retrieve` method, and mypy verifies that statically at the call site. The interface lives in what the *consumer* expects, not what the *implementation* announces. This makes retrofitting an interface onto existing code free, which is why you'll define protocols more liberally here than you would in C#.

#### 2. Configuration, and the index key

`coderag/config.py`:

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    repo_path: Path
    # chunking
    max_chunk_tokens: int = 400
    merge_below_tokens: int = 25
    include_header: bool = True
    include_doc_comments: bool = True
    include_type_summaries: bool = True
    # embedding
    embedding_model: str = "jinaai/jina-embeddings-v2-base-code"
    # retrieval
    retrieve_k: int = 50
    rerank_keep: int = 12
    bm25_weight: float = 0.5
    rrf_k: int = 60
    # generation
    generator_model: str = "claude-sonnet-4-6"
    temperature: float = 0.0

    model_config = {"env_file": ".env", "env_prefix": "CODERAG_"}
```

Everything that Week 3 sweeps must be here. If a value is hardcoded in a function, you cannot sweep it, and you will end up editing source between experiment runs — which is how people lose track of what produced which number.

Now the piece that prevents the single most common experimental disaster:

```python
INDEX_AFFECTING = {
    "max_chunk_tokens", "merge_below_tokens", "include_header",
    "include_doc_comments", "include_type_summaries", "embedding_model",
}

@property
def index_key(self) -> str:
    payload = json.dumps(
        {k: getattr(self, k) for k in sorted(INDEX_AFFECTING)},
        sort_keys=True,
    )
    return hashlib.sha256(payload.encode()).hexdigest()[:12]

@property
def collection_name(self) -> str:
    return f"code_{self.index_key}"
```

Change any parameter that affects the index and you automatically target a different collection. Without this, you will run a twelve-configuration sweep on Day 11, write all twelve into the same collection, and get twelve nearly identical results that you'll spend an afternoon failing to explain.

#### 3. CLI

```bash
coderag index [--force] [--incremental]
coderag ask "question" [--k 15] [--no-test] [--project Ordering.Application]
coderag stats
coderag config          # prints resolved settings + index_key
```

`coderag config` matters more than it looks. When a result surprises you three weeks from now, the first question is always "what configuration produced this?"

#### 4. Clean up

Delete the `scripts/` one-offs the CLI supersedes. Get `uv run mypy --strict coderag/` clean. Commit.

### Consolidation (1–1.5 hrs)

1. **Verify protocol conformance at runtime** once, in a test: `assert isinstance(LocalEmbedder(...), Embedder)`. `@runtime_checkable` only checks method *names*, not signatures, so this is a smoke test rather than a proof — mypy does the real work.
2. **Re-index under the new config system**, confirm the collection name contains the hash, and confirm the chunk count matches Week 1's.
3. Write `ARCHITECTURE.md`: the pipeline stages, which protocol each stage satisfies, and where the seams are. One page. You'll extend it weekly and it's most of Day 25's README.

### Reading (30 min)

PEP 544 (structural subtyping) and the mypy docs on Protocol. Focus on the difference between nominal and structural typing and when each is preferable.

### ✅ Done when

- `coderag ask "..."` works through the CLI
- `coderag config` prints settings with a stable `index_key`
- `mypy --strict` clean
- `ARCHITECTURE.md` exists

---

## Day 7 — Tue 4 Aug · Tracing and citations

**What you're learning today:** that you cannot debug a retrieval system from its answers, only from its intermediate state.

### Hard block (2 hrs)

#### 1. Spans

```python
@dataclass
class Span:
    name: str
    start: float
    end: float | None = None
    attrs: dict[str, Any] = field(default_factory=dict)

    @property
    def duration_ms(self) -> float:
        return (self.end - self.start) * 1000 if self.end else 0.0

@contextmanager
def span(trace: "Trace", name: str, **attrs: Any) -> Iterator[Span]:
    s = Span(name=name, start=time.perf_counter(), attrs=attrs)
    trace.spans.append(s)
    try:
        yield s
    finally:
        s.end = time.perf_counter()
```

Instrument: `embed_query`, `vector_search`, `bm25_search`, `fuse`, `rerank`, `build_prompt`, `generate`. Several don't exist yet — add the spans as you add the stages.

#### 2. The trace record

```python
class RetrievedRef(BaseModel):
    chunk_id: str
    file_path: str          # ← the field the eval harness reads
    symbol_name: str | None
    start_line: int
    end_line: int
    score: float
    rank: int
    source: str             # "vector" | "bm25" | "fused" | "reranked"

class Trace(BaseModel):
    trace_id: str
    timestamp: datetime
    query: str
    index_key: str
    git_sha: str
    config_snapshot: dict[str, Any]
    retrieved: list[RetrievedRef]
    prompt_tokens: int
    completion_tokens: int
    answer: str
    citations: list[Citation]
    spans: list[dict[str, Any]]
```

**`file_path` on every retrieved reference is the load-bearing field.** Days 9 onward compute retrieval quality by comparing these paths against ground truth. Without it you'd need an LLM judge for something that's a string comparison.

Record `source` too. On Day 14 you'll want to know whether a chunk arrived via BM25 or the vector store, and reconstructing that later is impossible.

Append one JSON object per query to `traces/YYYY-MM-DD.jsonl`. Add `coderag trace show <id>` that pretty-prints a trace with the retrieved chunks in full — you will run this hundreds of times.

#### 3. Citations

Update the prompt so chunks are numbered and every claim carries `[n]`. Then parse:

```python
CITATION_RE = re.compile(r"\[(\d+(?:\s*,\s*\d+)*)\]")

def parse_citations(answer: str, chunks: list[ScoredChunk]) -> list[Citation]:
    seen: dict[int, Citation] = {}
    for match in CITATION_RE.finditer(answer):
        for part in match.group(1).split(","):
            idx = int(part.strip())
            if not (1 <= idx <= len(chunks)):
                continue                      # model invented an index
            if idx in seen:
                continue
            c = chunks[idx - 1].chunk
            seen[idx] = Citation(
                index=idx, file_path=c.file_path,
                start_line=c.start_line, end_line=c.end_line,
                symbol_name=c.symbol_name,
            )
    return [seen[k] for k in sorted(seen)]
```

Handle all four failure modes explicitly: no citations at all, out-of-range index, comma-grouped `[1,3]`, and repeated indices. The model produces all of them.

Render a sources footer as `path:start-end` so it's clickable in most terminals and pasteable into an IDE.

### Consolidation (1–1.5 hrs)

1. **Re-run Day 5's 20 questions**, now traced. Read all 20 traces line by line — the actual retrieved source, not the ids.
2. **Re-classify every failure** now that you can see what was retrieved. Your Week 1 classifications were guesses; these aren't.
3. Unit-test the citation parser against every malformed form.
4. **Measure citation rate:** what fraction of answers cite anything? Under 80% means the prompt isn't insisting hard enough.

### Reading (30 min)

Observability for LLM applications — span hierarchies, what's worth recording, and why sampling traces is a mistake at this scale (record everything; you have hundreds of queries, not millions).

### ✅ Done when

- Every query writes a complete trace with `file_path` on each retrieval
- `coderag trace show <id>` is readable
- Citation parser tests green including malformed inputs
- 20 failures re-classified from evidence

---

## Day 8 — Wed 5 Aug · The golden set, part one

**What you're learning today:** what makes an evaluation question useful, by writing every one yourself.

### Hard block (2 hrs)

#### 1. The case model

```python
class GoldenCase(BaseModel):
    id: str
    question: str
    kind: Literal["symbol", "behaviour", "cross_cutting",
                  "trace", "config", "usage", "unanswerable"]
    expected_files: list[str]      # repo-relative — the ground truth
    expected_symbols: list[str]
    ideal_answer: str
    difficulty: Literal["easy", "medium", "hard"]
    notes: str | None = None
```

`expected_files` is what makes this whole plan work. For a prose corpus, "was the right context retrieved?" needs a judgement call. For code it's a set comparison against file paths — exact, free, instant, and reproducible.

#### 2. The taxonomy

Write these definitions into `evals/README.md` precisely enough that you'd tag consistently in a month.

| Kind | What it tests | Example |
|---|---|---|
| **symbol** | Direct lookup by name or concept | "Where is the retry policy configured?" |
| **behaviour** | What happens in a scenario | "What happens when order validation fails?" |
| **cross_cutting** | Set membership across files | "Which classes implement `IOrderRepository`?" |
| **trace** | Call relationships | "What calls `SendConfirmationEmail`?" |
| **config** | Settings, DI wiring, environment | "How is the HTTP client for the payments API registered?" |
| **usage** | How to use something, often answered by tests | "How do I construct a valid `CreateOrderCommand`?" |
| **unanswerable** | Plausible, genuinely absent | "Where is the GraphQL schema defined?" |

#### 3. Write 25 by hand

**No LLM today.** Writing them yourself is how you learn what your repo actually contains, and hand-written questions are phrased the way a human would ask — which is the distribution you're optimising for.

Target distribution:

| Kind | Count |
|---|---|
| symbol | 5 |
| behaviour | 5 |
| cross_cutting | 4 |
| trace | 3 |
| config | 3 |
| usage | 2 |
| unanswerable | 3 |

Rules while writing:

- **Phrase them as a colleague would.** "Where does the discount get applied?" not "Locate the method responsible for discount application." You're evaluating against real usage.
- **Fill `expected_files` by actually looking.** Open the repo, find the files, paste the paths. Guessing here poisons every measurement for five weeks.
- **List every acceptable file.** If the answer legitimately spans three files, list three. Recall is computed against the set.
- **For `unanswerable`, leave `expected_files` empty** and make the question genuinely plausible. "Where's the Kafka consumer?" in a repo with no Kafka is a good case. "Where's the quantum module?" is not — no system would be fooled.

### Consolidation (1–1.5 hrs)

1. **Loader with pydantic validation** that fails loudly on a malformed row, and additionally asserts every path in `expected_files` exists in the repo. Typos here are silent and destructive.
2. **Re-read your 25 questions cold.** Any you can't answer yourself without opening the repo is either too hard or badly worded. Fix or cut.
3. Commit as `evals/golden_v1.jsonl`.

### Reading (30 min)

RAGAS documentation — read what each of the four metrics computes *before* you run them on Friday.

### ✅ Done when

- 25 validated cases load, every `expected_files` path verified to exist
- Taxonomy documented in `evals/README.md`
- The unanswerable cases are genuinely plausible

---

## Day 9 — Thu 6 Aug · Golden set part two, and the metric that matters

**What you're learning today:** classical IR metrics, and why having a fast free deterministic one changes how you work.

### Hard block (2 hrs)

#### 1. Scale to 60

`scripts/draft_cases.py` — sample chunks, have the model draft candidate questions. Two prompting strategies, and you want both:

- *"What question would this code answer?"* — produces symbol and behaviour questions
- *"What would a new developer ask that this code answers?"* — produces usage and config questions, phrased more naturally

Generate 70 candidates. Then hand-review every single one.

**Expect to delete around 40%.** The dominant failure: the model writes a question answerable from the chunk's own text, which tests nothing about retrieval. "What does `CreateOrderCommandHandler.Handle` do?" contains the answer's location in the question. Cut those or rewrite them to describe the behaviour without naming the symbol.

Reach 60 total.

#### 2. The retrieval metrics

`evals/metrics.py`:

```python
def hit_rate_at_k(retrieved: list[str], expected: list[str], k: int) -> float:
    """1.0 if any expected file appears in the top k."""
    return float(bool(set(retrieved[:k]) & set(expected)))

def recall_at_k(retrieved: list[str], expected: list[str], k: int) -> float:
    """Fraction of expected files found in top k."""
    if not expected:
        return 1.0                      # unanswerable: nothing to find
    return len(set(retrieved[:k]) & set(expected)) / len(set(expected))

def mrr(retrieved: list[str], expected: list[str]) -> float:
    """1/rank of the first correct file."""
    for i, path in enumerate(retrieved, start=1):
        if path in expected:
            return 1.0 / i
    return 0.0
```

De-duplicate file paths before scoring — several chunks from one file should count once.

**These metrics need no LLM.** They run in under a second over all 60 cases, cost nothing, and give identical results every time. This is the single biggest practical advantage a code corpus gives you, and it should change your working rhythm: run this after *every* change, dozens of times a day. Save the expensive LLM-judged metrics for once-a-day confirmation.

#### 3. The fast eval loop

```bash
make retrieval-eval     # 60 cases, ~10s, free, deterministic
```

Output a table broken down **by question kind**, not just an aggregate:

```
kind            n   hit@10   recall@10   MRR
symbol         12    0.92      0.85      0.71
behaviour      12    0.67      0.54      0.38
cross_cutting  10    0.30      0.18      0.15
trace           8    0.25      0.14      0.11
config          8    0.75      0.69      0.52
usage           6    0.67      0.58      0.44
unanswerable    4      —         —         —
```

The aggregate hides everything. The per-kind view tells you where to spend Week 3, and it makes the `cross_cutting` and `trace` weakness — which Week 4 exists to fix — impossible to ignore.

### Consolidation (1–1.5 hrs)

1. **Coverage check.** Which projects or top-level folders have zero questions? Hand-write cases for the gaps. An eval set that ignores half the repo will happily tell you a change is fine when it broke that half.
2. **Freeze and version.** `evals/golden_v1.jsonl`, committed. If you later add cases, it becomes `v2` and you note which runs used which — comparing across eval-set versions is meaningless and easy to do by accident.
3. **Record the pre-baseline** from the fast metrics. This is Week 1's system, measured properly for the first time.

### Reading (30 min)

Classical IR evaluation: precision, recall, MRR, NDCG. Understand what NDCG adds over MRR (graded relevance and position discounting) and why you're not using it yet — your ground truth is binary.

### ✅ Done when

- 60 human-reviewed cases, all paths verified
- `make retrieval-eval` runs in under 15 seconds and prints a per-kind table
- Coverage gaps closed
- Baseline recorded

---

## Day 10 — Fri 7 Aug · RAGAS, the full baseline, and the comparison harness

**What you're learning today:** what LLM-as-judge metrics measure, where they're unreliable, and how to build a change-tracking discipline.

### Hard block (2 hrs)

#### 1. Wire RAGAS

```bash
uv add ragas datasets
```

Run the full pipeline over all 60 cases, collecting question, answer, retrieved contexts, and ground-truth answer. Compute faithfulness, answer relevancy, context precision, context recall.

**An honest caveat, and it matters.** RAGAS metrics are LLM-as-judge underneath, and judge models are meaningfully weaker at evaluating code than prose. Faithfulness scoring works by decomposing an answer into claims and checking each against the context — that decomposition is noisier when the claims are about control flow and type relationships. Expect run-to-run variance of ±0.03 on identical inputs.

The practical consequence:

| Metric | Use for | Cadence |
|---|---|---|
| `hit@k`, `recall@k`, MRR | Every retrieval experiment | Constantly — it's free |
| RAGAS faithfulness | Generation and prompt changes | Once daily |
| RAGAS context precision/recall | Cross-check on the fast metrics | Weekly |

Treat file-path recall as your primary instrument and RAGAS as the secondary one. This inverts the usual advice, and it's correct here because your ground truth is objective.

#### 2. Persist runs

```python
class EvalRun(BaseModel):
    run_id: str
    timestamp: datetime
    git_sha: str
    index_key: str
    config_snapshot: dict[str, Any]
    golden_set_version: str
    retrieval_metrics: dict[str, float]
    retrieval_by_kind: dict[str, dict[str, float]]
    ragas_metrics: dict[str, float] | None
    per_case: list[dict[str, Any]]
```

Save to `evals/runs/2026-08-07-<sha>-<index_key>.json`. The config snapshot is non-negotiable — a score without the configuration that produced it is a number you can't act on.

`per_case` matters more than you'd think. Aggregate deltas tell you *whether* something changed; per-case deltas tell you *what* changed, and often a change that moves the aggregate by +0.01 has actually fixed six cases and broken five. That's a very different situation from a uniform small gain.

#### 3. The comparison harness

```python
def compare(baseline: EvalRun, current: EvalRun) -> None:
    # aggregate table with deltas, flagging moves > 0.05
    # per-kind table
    # regressions: cases that passed in baseline and fail now
    # fixes: cases that failed in baseline and pass now
```

The regression and fix lists are the most useful output. Read the names.

```bash
make retrieval-eval   # fast, free, every change
make eval             # full, with RAGAS, daily
make compare BASE=<run_id>
```

#### 4. Run the baseline

Record it. Everything for the next three weeks is measured against this number.

### Consolidation (1–1.5 hrs)

**Write-up #2:** *"What my code RAG system actually scores — the unflattering baseline."*

Publish the per-kind table. The `cross_cutting` and `trace` rows will be poor, and saying so plainly — along with why similarity search structurally cannot answer "what calls this" — is the most interesting thing in the post.

Then read the RAGAS source for `faithfulness`. Find the prompt it uses for claim decomposition. Once you've seen it, you'll understand exactly why the metric is noisy on code, and you'll trust it the right amount.

### Reading (30 min)

LLM-as-judge: position bias, verbosity bias, self-preference, and the calibration literature. You're about to spend three weeks optimising partly against a judge model, so know what it rewards that you don't care about.

### ✅ Done when

- `make eval` produces retrieval metrics, per-kind breakdown, and RAGAS scores
- Baseline run saved with full config snapshot
- `make compare` shows deltas, regressions, and fixes by name
- Write-up published

---

## Week 2 retrospective

You should now be able to explain:

- Why structural typing makes retrofitting interfaces cheaper than in C#
- Why a config hash must drive collection naming
- What hit rate, recall@k, and MRR each measure and when they disagree
- Why file-path recall beats LLM-judged context recall for a code corpus
- What LLM-as-judge metrics are actually doing, and where they're weak

**What you've built:** a packaged, traced, fully configurable system with a 60-case golden set, an instant free retrieval metric, and a recorded baseline.

**What's still broken, deliberately:** retrieval is vector-only. No lexical search, no reranking, no structural awareness. Your `cross_cutting` and `trace` scores are poor and you know exactly how poor.

Week 3 fixes retrieval. Every change from here gets a number.
