# Machine Setup

Everything you need installed and verified before Day 1. Budget 60–90 minutes, most of it downloads.

Covers three paths: **Windows + WSL2** (recommended for Windows), **Windows native** (viable, with caveats), and **macOS**. Skip to yours, then do the [Common Setup](#common-setup) and [Verification](#verification) sections, which apply to all three.

---

## Hardware

| | Minimum | Comfortable |
|---|---|---|
| RAM | 16 GB | 32 GB |
| Free disk | 25 GB | 50 GB |
| CPU cores | 4 physical | 8+ physical |
| GPU | None required | Nice, not needed |

The RAM figure isn't padding. By Week 4 you'll have an embedding model, a cross-encoder reranker, a Qdrant container, and Docker Desktop resident simultaneously. On 8 GB you'll be swapping.

**No GPU is required.** Everything in the plan runs on CPU. A GPU makes indexing faster and changes nothing else. Apple Silicon gets a meaningful speedup via MPS — see [Performance tuning](#performance-tuning).

### What to expect on time

Rough indexing throughput for an 8,000-chunk index with `jina-embeddings-v2-base-code`:

| Machine | Full index | Incremental (1 file) |
|---|---|---|
| 4-core CPU | 8–15 min | ~2 s |
| 8-core CPU | 4–8 min | ~2 s |
| Apple Silicon (MPS) | 1–3 min | ~1 s |

Week 3 runs a dozen full re-indexes in a day. If you're on the slow end, run the sweeps against a 30-case subset and confirm the winner on the full 60 — the plan flags this where it matters.

---

## Which path

### Windows: WSL2 or native?

Honest answer: **both work.** The Python AI ecosystem has decent Windows wheels now, and `uv`, `tree-sitter`, `torch`, and `chromadb` all install natively without drama.

The real tradeoff is about where your C# repo lives.

| | WSL2 | Windows native |
|---|---|---|
| Ecosystem parity | Best — everything is Linux-first | Good, occasional rough edges |
| Docker / Qdrant (Week 4) | Native, fast | Via Docker Desktop, fine |
| Reading a repo on `C:\` | **Slow** — see below | Fast |
| `trust_remote_code` models | No issues | Rare posix assumptions |
| Matches the plan's commands | Exactly | Minor translation needed |

**The filesystem boundary is the deciding factor.** WSL2 reaching across to `/mnt/c/...` goes through a network protocol, and it is dramatically slower for the kind of work Day 1 does — walking thousands of files and reading each one. A repo walk that takes 3 seconds inside WSL can take 60+ seconds across the boundary.

So:

- **Your repo can live inside WSL2** (clone a second copy at `~/repos/YourSolution`, pull to refresh) → **use WSL2**. Cleanest path, matches every command in the plan verbatim.
- **You need to index the working copy you build in Visual Studio, in place** → **use Windows native**. Avoiding the boundary is worth more than ecosystem parity.

A second copy inside WSL is usually fine — you're reading it, not building it. Week 5's git-aware indexing works off `git diff`, so a `git pull` before re-indexing keeps it current.

### macOS

One path, no decisions. Apple Silicon or Intel both work; Apple Silicon is noticeably faster.

---

## Windows + WSL2

### 1. Install WSL2

In PowerShell **as Administrator**:

```powershell
wsl --install -d Ubuntu-24.04
```

Reboot when prompted. On first launch you'll create a Linux username and password — unrelated to your Windows account.

Verify:

```powershell
wsl -l -v
```

You want `VERSION 2`. If it says 1:

```powershell
wsl --set-version Ubuntu-24.04 2
wsl --set-default-version 2
```

**If `wsl --install` fails**, virtualisation is likely disabled in BIOS/UEFI. Look for Intel VT-x, AMD-V, or SVM Mode. On some laptops it's under a Security or Advanced menu.

### 2. Cap WSL2 resources

WSL2 will otherwise claim most of your RAM and not give it back. Create `C:\Users\<you>\.wslconfig`:

```ini
[wsl2]
memory=16GB
processors=8
swap=8GB
localhostForwarding=true
```

Set `memory` to roughly half your physical RAM, `processors` to your physical core count. Then `wsl --shutdown` in PowerShell and reopen your Ubuntu terminal.

### 3. Exclude WSL from Defender

Real-time scanning of the WSL virtual disk costs meaningful performance on file-heavy work — which is exactly what Day 1 does.

In Windows Security → Virus & threat protection → Manage settings → Exclusions, add:

- `\\wsl$` (folder)
- Your Ubuntu VHDX, typically under `%LOCALAPPDATA%\Packages\CanonicalGroupLimited.Ubuntu*\LocalState\`

Your call on the security tradeoff. It's a development VM, not a download folder.

### 4. Base packages

Inside the Ubuntu terminal:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential curl git pkg-config libssl-dev unzip
```

`build-essential` matters — some Python packages compile C extensions if no wheel matches your platform.

### 5. Docker Desktop

Install [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop/). During setup, choose the **WSL2 backend**.

Then Settings → Resources → WSL Integration → enable for `Ubuntu-24.04`.

Verify from inside Ubuntu:

```bash
docker run --rm hello-world
```

You need this from Week 4 (Day 20, Qdrant). Install it now so the reboot doesn't eat a working session.

> Docker Desktop is free for personal use and small businesses. If licensing is an issue, [Rancher Desktop](https://rancherdesktop.io/) or Podman both work — you only need to run one container.

### 6. Clone your repo inside WSL

```bash
mkdir -p ~/repos && cd ~/repos
git clone <your-repo-url> YourSolution
```

Then, still inside WSL:

```bash
git config --global core.autocrlf false
```

Without this, git injects Windows line endings into files inside the Linux filesystem, and your chunk boundaries and content hashes get noisy for no reason.

Now skip to [Common Setup](#common-setup).

---

## Windows native

Use this if your repo must stay on `C:\` and be indexed in place.

### 1. Python and uv

Install `uv` in PowerShell:

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

Close and reopen the terminal, then:

```powershell
uv --version
uv python install 3.12
```

`uv` manages the Python install itself. Don't install Python from python.org or the Microsoft Store — you'll end up with PATH ambiguity.

### 2. Build tools

Some packages compile from source when no wheel matches. Install [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/) and select **Desktop development with C++**.

You may never need it. When you do, the error is a wall of compiler output and it's much easier to have installed it already.

### 3. Git

If you don't already have it: [git-scm.com](https://git-scm.com/download/win). During install, choose "Checkout as-is, commit as-is" to avoid line ending rewriting.

### 4. Docker Desktop

Same as above — install it now, needed from Day 20.

### 5. Translation notes

The plan's commands assume a POSIX shell. Differences you'll hit:

| Plan says | Windows PowerShell |
|---|---|
| `source .venv/bin/activate` | `.venv\Scripts\Activate.ps1` |
| `export FOO=bar` | `$env:FOO = "bar"` |
| `~/coderag` | `$HOME\coderag` |
| `make eval` | Install `make` via `winget install GnuWin32.Make`, or use `uv run python -m evals.run` |

Paths in your code should use `pathlib` and `.as_posix()` throughout — the plan does this from Day 1 specifically so that repo-relative paths stay portable and your golden set's `expected_files` compare correctly.

If PowerShell blocks script execution:

```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned
```

Now go to [Common Setup](#common-setup).

---

## macOS

### 1. Xcode command line tools

```bash
xcode-select --install
```

Needed for compiling native extensions. It's a few GB and takes a while; start it first.

### 2. Homebrew

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Follow the printed instructions to add it to your PATH — on Apple Silicon it installs to `/opt/homebrew` and this step is easy to skip and then be confused by.

```bash
brew install git
```

### 3. Docker

```bash
brew install --cask docker
```

Launch it once from Applications to complete setup. Or, if you'd rather not run Docker Desktop:

```bash
brew install colima docker
colima start --cpu 4 --memory 8
```

Colima is lighter and works fine for a single Qdrant container.

Verify:

```bash
docker run --rm hello-world
```

### 4. A note on case sensitivity

APFS is case-**insensitive** by default. Two C# files differing only in case would collapse into one, and `git` will occasionally behave oddly around renames that only change case.

This is rare and usually harmless. If your repo has case-only collisions (check with `git ls-files | sort -f | uniq -di`), consider a case-sensitive APFS volume for it.

Now go to [Common Setup](#common-setup).

---

## Common Setup

Applies to all three paths. Commands shown for POSIX shells; translate for PowerShell using the table above.

### 1. Install uv

Skip if you did it in the Windows native section.

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
uv --version
```

`uv` replaces pip, venv, pyenv, and pip-tools. It's dramatically faster than pip, which matters when you're rebuilding environments and installing torch.

### 2. Scaffold the project

```bash
mkdir -p ~/coderag && cd ~/coderag
uv init --python 3.12
uv venv
source .venv/bin/activate
```

**Why 3.12 and not 3.13?** ML wheels lag the newest Python by a few months, and hitting a missing wheel that forces a source build is a bad way to spend Monday morning. 3.12 is boring and everything supports it.

### 3. Install torch — CPU only

This step saves you 2+ GB and several minutes, and almost everyone misses it.

On Linux, `pip install torch` pulls the CUDA build by default, dragging in a pile of NVIDIA packages you have no use for on a CPU machine.

Add to `pyproject.toml`:

```toml
[[tool.uv.index]]
name = "pytorch-cpu"
url = "https://download.pytorch.org/whl/cpu"
explicit = true

[tool.uv.sources]
torch = { index = "pytorch-cpu" }
```

Then:

```bash
uv add torch
```

> `uv`'s index configuration syntax has changed across releases. If this errors, check `uv` docs for the current form, or fall back to `uv pip install torch --index-url https://download.pytorch.org/whl/cpu`.

**On macOS this is unnecessary** — the default macOS wheel has no CUDA. Just `uv add torch`.

Verify:

```bash
uv run python -c "import torch; print(torch.__version__, torch.backends.mps.is_available() if hasattr(torch.backends,'mps') else 'no mps')"
```

### 4. Core dependencies

```bash
uv add pydantic pydantic-settings anthropic python-dotenv tiktoken \
       pathspec rich typer
uv add tree-sitter tree-sitter-c-sharp
uv add sentence-transformers chromadb einops
uv add --dev mypy ruff pytest
```

Later weeks add `rank-bm25`, `qdrant-client`, `ragas`, `fastapi`, `networkx` and others. The plan installs them when they're first used.

### 5. Pre-download the models

Do this now, over coffee, rather than in the middle of Day 4.

```bash
uv run python - <<'PY'
from sentence_transformers import SentenceTransformer, CrossEncoder
print("embedding model...")
SentenceTransformer("jinaai/jina-embeddings-v2-base-code", trust_remote_code=True)
print("control model...")
SentenceTransformer("BAAI/bge-small-en-v1.5")
print("reranker...")
CrossEncoder("BAAI/bge-reranker-base")
print("done")
PY
```

Roughly 2 GB total, cached in `~/.cache/huggingface`. The reranker isn't needed until Day 15 but you may as well pull it now.

`trust_remote_code=True` means the model ships custom Python that executes on load. It's a well-known model from a reputable publisher, but you should know that's what the flag does rather than clicking past it.

### 6. API key

```bash
echo ".env" > .gitignore
printf "\n.venv/\nchroma/\ntraces/\nqdrant_storage/\n__pycache__/\n*.pyc\n" >> .gitignore
git init && git add .gitignore && git commit -m "gitignore"

echo "ANTHROPIC_API_KEY=sk-ant-..." > .env
```

**Commit the `.gitignore` before the `.env` exists.** This ordering is the single most common way keys reach GitHub.

Get a key at [console.anthropic.com](https://console.anthropic.com). Add $20 — across five weeks, mostly eval runs, you'll spend somewhere in the $15–30 range. Set a spend limit while you're in there.

### 7. VS Code

Install from [code.visualstudio.com](https://code.visualstudio.com).

**WSL2 users:** install VS Code on *Windows*, then the Remote-WSL extension. Open projects by running `code .` from your Ubuntu terminal — not by browsing to `\\wsl$` from Explorer. Extensions must then be installed on the WSL side; VS Code prompts you.

Extensions:

```bash
code --install-extension ms-python.python
code --install-extension charliermarsh.ruff
code --install-extension ms-python.mypy-type-checker
code --install-extension ms-azuretools.vscode-docker
```

---

## Verification

Save as `scripts/verify_setup.py` and run it. It checks everything at once and tells you exactly what's missing.

```python
"""Pre-flight check. Run: uv run python scripts/verify_setup.py"""
import platform, shutil, subprocess, sys, os
from pathlib import Path

OK, FAIL = "\033[92m✓\033[0m", "\033[91m✗\033[0m"
results: list[tuple[bool, str, str]] = []

def check(name: str, fn) -> None:
    try:
        results.append((True, name, str(fn())))
    except Exception as e:
        results.append((False, name, f"{type(e).__name__}: {e}"))

check("platform", lambda: f"{platform.system()} {platform.machine()}")

def _py():
    assert sys.version_info[:2] == (3, 12), f"want 3.12, got {sys.version_info[:2]}"
    return platform.python_version()
check("python 3.12", _py)

def _torch():
    import torch
    dev = "cpu"
    if hasattr(torch.backends, "mps") and torch.backends.mps.is_available():
        dev = "mps"
    elif torch.cuda.is_available():
        dev = "cuda"
    return f"{torch.__version__} (device: {dev}, threads: {torch.get_num_threads()})"
check("torch", _torch)

def _embed():
    from sentence_transformers import SentenceTransformer
    m = SentenceTransformer("jinaai/jina-embeddings-v2-base-code", trust_remote_code=True)
    v = m.encode(["public async Task<Guid> Handle(CreateOrderCommand cmd)"])
    return f"dim={v.shape[1]}"
check("embedding model", _embed)

def _ts():
    import tree_sitter_c_sharp as tscs
    from tree_sitter import Language, Parser
    src = b"namespace Foo;\npublic class Bar { public int Baz() => 1; }"
    tree = Parser(Language(tscs.language())).parse(src)
    assert tree.root_node.type == "compilation_unit"
    kinds = {n.type for n in tree.root_node.children}
    assert "file_scoped_namespace_declaration" in kinds, kinds
    return "parsed, file-scoped namespace OK"
check("tree-sitter c#", _ts)

def _tok():
    import tiktoken
    return f"{len(tiktoken.get_encoding('cl100k_base').encode('public class Foo {}'))} tokens"
check("tiktoken", _tok)

def _chroma():
    import chromadb, tempfile
    with tempfile.TemporaryDirectory() as d:
        c = chromadb.PersistentClient(path=d)
        col = c.get_or_create_collection("t", metadata={"hnsw:space": "cosine"})
        col.upsert(ids=["1"], embeddings=[[0.1] * 8], documents=["x"])
        assert col.count() == 1
    return "read/write OK"
check("chromadb", _chroma)

def _api():
    from dotenv import load_dotenv
    from anthropic import Anthropic
    load_dotenv()
    key = os.environ.get("ANTHROPIC_API_KEY")
    assert key and key.startswith("sk-ant-"), "ANTHROPIC_API_KEY missing or malformed"
    r = Anthropic().messages.create(
        model="claude-sonnet-4-6", max_tokens=10,
        messages=[{"role": "user", "content": "Reply with the word: ready"}])
    return f"{r.content[0].text.strip()} ({r.usage.input_tokens} in)"
check("anthropic api", _api)

def _docker():
    assert shutil.which("docker"), "docker not on PATH"
    subprocess.run(["docker", "info"], capture_output=True, check=True, timeout=30)
    return "daemon running"
check("docker", _docker)

def _disk():
    free_gb = shutil.disk_usage(Path.home()).free / 1e9
    assert free_gb > 20, f"only {free_gb:.1f} GB free, want 20+"
    return f"{free_gb:.0f} GB free"
check("disk space", _disk)

def _mem():
    import os
    try:
        gb = os.sysconf("SC_PAGE_SIZE") * os.sysconf("SC_PHYS_PAGES") / 1e9
    except (ValueError, AttributeError):
        return "unknown (non-posix)"
    assert gb > 14, f"only {gb:.0f} GB RAM detected"
    return f"{gb:.0f} GB"
check("memory", _mem)

print()
for ok, name, detail in results:
    print(f"  {OK if ok else FAIL} {name:22} {detail}")
failed = [n for ok, n, _ in results if not ok]
print()
if failed:
    print(f"\033[91m{len(failed)} check(s) failed:\033[0m {', '.join(failed)}")
    sys.exit(1)
print("\033[92mAll checks passed. Ready for Day 1.\033[0m")
```

Every line should be green before Monday. The `tree-sitter c#` check specifically asserts that file-scoped namespaces parse — that's the Week 1 gotcha that silently strips namespace metadata from every chunk in a modern codebase.

---

## Performance tuning

### Thread counts

`torch` defaults to using every logical core, which for embedding workloads is usually slower than using physical cores only — hyperthreads contend for the same vector units.

```python
import torch
torch.set_num_threads(8)   # your physical core count
```

Or via environment:

```bash
export OMP_NUM_THREADS=8
export TOKENIZERS_PARALLELISM=false   # silences a noisy HF warning
```

Benchmark both. On some machines the difference is 30%.

### Apple Silicon: use MPS

This is a real speedup and it's one line:

```python
model = SentenceTransformer("jinaai/jina-embeddings-v2-base-code",
                            trust_remote_code=True, device="mps")
```

Typically 3–5× faster than CPU for embedding. Some operations fall back to CPU silently, so **benchmark rather than assume** — and if you hit an unsupported-op error, `PYTORCH_ENABLE_MPS_FALLBACK=1` routes those to CPU rather than crashing.

Make the device a config value from Day 6 so you can compare.

### WSL2: keep work off `/mnt/c`

Worth repeating because it's the most common WSL2 performance complaint and it's entirely avoidable. Your project, your venv, your model cache, and ideally your repo copy should all live under `~/`. Only reach across to `/mnt/c` if you genuinely have no alternative.

### Docker resources

Docker Desktop → Settings → Resources. Qdrant is modest — 2 GB and 2 CPUs is plenty for this corpus size. Don't let Docker claim resources the embedding model needs.

---

## Troubleshooting

| Symptom | Cause and fix |
|---|---|
| `uv: command not found` after install | Shell PATH not reloaded. `source $HOME/.local/bin/env`, or open a new terminal. |
| `wsl --install` fails | Virtualisation disabled in BIOS. Look for VT-x / AMD-V / SVM Mode. |
| WSL shows VERSION 1 | `wsl --set-version <distro> 2` then `wsl --set-default-version 2`. |
| Torch install pulls ~2.5 GB of NVIDIA packages | You skipped the CPU index config. Remove torch and redo step 3. |
| `trust_remote_code` prompt or error | Expected — the model ships custom code. Pass `trust_remote_code=True` explicitly. |
| `Parser() takes no arguments` | tree-sitter version difference. Use `p = Parser(); p.set_language(LANG)`. |
| tree-sitter returns empty captures | Query API changed shape across versions — dict vs list of tuples. Check in the REPL. |
| Repo walk takes minutes on WSL2 | You're reading across `/mnt/c`. Clone inside WSL. |
| `docker: command not found` in WSL | Docker Desktop → Settings → Resources → WSL Integration, enable your distro. |
| Model download stalls | HF hub can be flaky. `export HF_HUB_ENABLE_HF_TRANSFER=1` after `uv add hf-transfer`. |
| Embedding is very slow on Apple Silicon | You're on CPU. Pass `device="mps"`. |
| VS Code can't see the venv (WSL2) | You opened from Explorer. Close, run `code .` from the Ubuntu terminal. |
| `mypy` floods with missing-stub errors | Add `ignore_missing_imports = true` under `[tool.mypy]`. |
| Non-ASCII characters break chunk offsets | You sliced a decoded string with byte offsets. Slice bytes, then decode. |

---

## Before Monday

A short checklist:

- [ ] `verify_setup.py` all green
- [ ] Docker installed and `docker run --rm hello-world` works
- [ ] Models pre-downloaded (~2 GB in `~/.cache/huggingface`)
- [ ] `.env` has a working API key, `.gitignore` committed first
- [ ] **Repository chosen and cloned** — 50k–500k lines, one you know well
- [ ] Line count noted: `git ls-files '*.cs' | xargs wc -l | tail -1`
- [ ] Primer read

That last item is the one that matters. Everything else here is installation; picking the right repo is the only decision on this page that you can't easily reverse.
