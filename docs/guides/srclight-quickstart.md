# Srclight Quickstart Guide

## What is Srclight?

**Srclight** is the **semantic layer** of the air-gapped AI coding architecture. Deep code indexing
for AI agents, built on SQLite FTS5, tree-sitter, embeddings, and the Model Context Protocol (MCP).

### Core Architecture

Srclight combines four indexing strategies into a single hybrid search engine:

1. **Tree-sitter parsing** extracts precise symbols (functions, classes, methods, interfaces, structs),
   builds call graph edges, and maps type hierarchies across 11 supported languages.
2. **FTS5 names index** — code-aware tokenization splitting `camelCase` and `::` operators.
3. **FTS5 trigram index** — substring matching across all source code.
4. **FTS5 docs index** — Porter stemming for natural language in docstrings.
5. **Semantic embeddings** — optional, via an OpenAI-compatible endpoint for concept-level queries.

Search results merge keyword and semantic matches through **Reciprocal Rank Fusion (RRF)**.

### Key Features

- **Hybrid Search** — FTS5 keyword + semantic embeddings via RRF
- **29 MCP Tools** — symbol search, relationship graphs, git intelligence, semantic search, build system awareness
- **Tree-sitter Symbol Extraction** — 11 languages, precise structural understanding
- **Relationship Graph** — callers, callees, blast radius, inheritance, test discovery
- **Git Change Intelligence** — blame, hotspots, WIP, commit history per symbol
- **Multi-repo Workspaces** — ATTACH across SQLite databases, UNION across schemas
- **Incremental Indexing** — content-hash detection, only re-indexes changed files
- **Auto-reindex** — git post-commit/post-checkout hooks
- **100% Offline** — single SQLite file per repo, no Docker, no cloud APIs

### Supported Languages

Python, C, C++, C#, JavaScript, TypeScript, PHP, Dart, Swift, Kotlin, Java, Go

## Installation

```bash
pip install srclight            # install latest (0.15.1+)
pip install --upgrade srclight  # upgrade if already installed
```

> **Version requirement**: OpenAI-compatible embedding support requires v0.11.0+. The current
> release is v0.15.1. Check your version with `pip show srclight`.

Optional extras:

```bash
pip install 'srclight[docs,pdf]'   # PDF, DOCX, XLSX, HTML, image extraction
pip install 'srclight[gpu]'         # GPU-accelerated vector search (CUDA 12.x)
pip install 'srclight[all]'         # Everything
```

### Air-gapped Preparation Checklist

> **Known issue**: `tree-sitter-dart` is listed as a dependency of srclight 0.15.1 but is not
> published on PyPI. Install srclight with `--no-deps` and add all other dependencies manually.
> Dart indexing is simply unavailable — irrelevant for C/C++ codebases.

Run the following **on a connected machine** before going air-gapped:

```bash
# 1. Download srclight wheel only (no deps — tree-sitter-dart is not on PyPI)
pip download "srclight==0.15.1" --no-deps -d ./srclight-wheels/

# 2. Download all dependencies except tree-sitter-dart
pip download \
  click mcp numpy \
  "tree-sitter>=0.21" \
  tree-sitter-c tree-sitter-cpp tree-sitter-c-sharp \
  tree-sitter-python tree-sitter-javascript tree-sitter-typescript \
  tree-sitter-go tree-sitter-java tree-sitter-kotlin \
  tree-sitter-rust tree-sitter-swift tree-sitter-php \
  tree-sitter-markdown \
  -d ./srclight-wheels/

# 3. Optionally include GPU or docs extras
pip download "srclight[gpu]==0.15.1" --no-deps -d ./srclight-wheels/

# 4. Transfer ./srclight-wheels/ to the air-gapped machine via approved media
```

Then on the **air-gapped machine**:

```bash
# 5. Install srclight + deps (ignoring tree-sitter-dart resolver warning)
pip install --no-index --find-links ./srclight-wheels/ --no-deps srclight==0.15.1
pip install --no-index --find-links ./srclight-wheels/ \
  click mcp numpy \
  tree-sitter \
  tree-sitter-c tree-sitter-cpp tree-sitter-c-sharp \
  tree-sitter-python tree-sitter-javascript tree-sitter-typescript \
  tree-sitter-go tree-sitter-java tree-sitter-kotlin \
  tree-sitter-rust tree-sitter-swift tree-sitter-php \
  tree-sitter-markdown

# Verify (resolver will warn about tree-sitter-dart — this is expected and harmless)
pip show srclight          # must show Version: 0.15.1
srclight --version         # must print srclight, version 0.15.1

# 6. Verify the internal embedding server is reachable
curl http://inference.internal/v1/models
```

## Configuration

**Srclight has no config file.** All configuration is via CLI flags at index time. The only
thing written to disk is `.srclight/index.db` (the SQLite database) plus optional embedding
sidecars (`.npy` files).

> `srclight index` automatically adds `.srclight/` to your `.gitignore`. The database and
> embedding files should never be committed.

## Quick Start Workflow

### Step 1: Index Your Project

```bash
# Index current directory (keyword search only)
cd /path/to/project
srclight index

# Index with embeddings via internal OpenAI-compatible server
# Model names starting with "text-embedding" are auto-detected as OpenAI-compatible:
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal srclight index \
  --embed openai:qwen3-embedding-8b

# For other model names (e.g. qwen3-embedding-8b), use the "openai:" prefix to
# force OpenAI-compatible provider — otherwise Srclight defaults to Ollama:
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal srclight index \
  --embed openai:qwen3-embedding-8b
```

Tree-sitter parses each file, extracts symbols and relationships, then the embedding server
generates vectors (if `--embed` is specified). Everything lands in `.srclight/index.db`.

### Step 2: Start the MCP Server

Single repo (stdio — one server per session):

```bash
srclight serve
```

For OpenCode / Claude Code integration:

```bash
claude mcp add srclight -- srclight serve
```

Workspace mode (SSE — persistent, recommended for multi-repo):

```bash
srclight serve --workspace myworkspace &
claude mcp add --transport sse srclight http://127.0.0.1:8742/sse
```

### Step 3: Search

```bash
# Keyword search
srclight search "authentication"

# Symbol-specific
srclight search --kind function "parseJson"
```

### Step 4: Incremental Updates

Re-running `srclight index` is incremental by default — only re-indexes files whose
content hash changed:

```bash
srclight index
```

Or install git hooks to keep the index fresh automatically:

```bash
srclight hook install
```

## Embedding Configuration

Embeddings are optional. Without `--embed`, you get keyword-only search (FTS5). With `--embed`,
you get hybrid search (FTS5 + semantic via RRF).

### Internal Server (Default for This Deployment)

```bash
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal srclight index \
  --embed openai:qwen3-embedding-8b
```

> **Model routing**: When `--embed` receives a value with the `openai:` prefix, Srclight uses the
> OpenAI-compatible provider. The base URL is set via `OPENAI_BASE_URL`. Any model
> served by the internal server works — `openai:qwen3-embedding-8b`, `openai:gte-large`, etc.

### Local Fallback: infinity-emb

If the internal server is unavailable, `infinity-emb` provides an identical OpenAI-compatible
embedding server running entirely on CPU.

**Install and run:**

```bash
pip install "infinity-emb[all]"

# Pre-download the model before going air-gapped
python -c "from huggingface_hub import snapshot_download; snapshot_download('Qwen/Qwen3-Embedding-8B')"
export HF_HUB_OFFLINE=1

# Run on port 7997
infinity_emb v2 --model-name-or-path Qwen/Qwen3-Embedding-8B --port 7997
```

**Index using the local server:**

```bash
OPENAI_BASE_URL=http://localhost:7997 srclight index --embed openai:qwen3-embedding-8b
# No OPENAI_API_KEY needed for local infinity-emb
```

### Embedding Backend Comparison

| Backend | Install | Memory | HTTP API | Air-Gap |
|---------|---------|--------|----------|---------|
| Internal server | None | N/A | Yes (OpenAI-compatible) | Yes (internal network) |
| `infinity-emb` | `pip install infinity-emb[all]` | ~600MB | Yes (OpenAI-compatible) | Yes (after model download) |
| `sentence-transformers` | `pip install sentence-transformers` | ~300MB | No (Python only) | Yes (after model download) |
| `fastembed` | `pip install fastembed` | ~200MB | No (Python only) | Yes (after model download) |

## Multi-Repo Workspaces

```bash
# Create a workspace
srclight workspace init myworkspace

# Add repos
srclight workspace add /path/to/repo1 -w myworkspace
srclight workspace add /path/to/repo2 -w myworkspace

# Index all repos with embeddings
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal \
  srclight workspace index -w myworkspace --embed openai:qwen3-embedding-8b

# Start MCP server in workspace mode
srclight serve --workspace myworkspace
```

## Use Cases

### Use Case 1: Finding Code by Description

```
You: Where do we handle JWT token validation?

→ srclight: hybrid_search("JWT token validation")

Found 5 matches:
1. src/auth/jwt.py:22 - validate_token()       score: 0.92
2. src/middleware/auth.py:15 - auth_middleware() score: 0.87
3. src/services/session.py:45 - check_session_valid() score: 0.78
```

### Use Case 2: Blast Radius Analysis

```
You: What breaks if I change validate_token?

→ srclight: get_dependents("validate_token", transitive=true)

Direct callers: 4 symbols
Transitive dependents: 23 symbols across 8 files
```

### Use Case 3: Git Intelligence

```
You: What changed recently in auth?

→ srclight: recent_changes(10)
→ srclight: git_hotspots(5)

Most changed files (last 30 days):
1. src/auth/jwt.py — 14 commits (bug magnet)
2. src/middleware/auth.py — 9 commits
```

### Use Case 4: Multi-repo Search

```bash
# Search across all repos simultaneously
srclight workspace search "user authentication" -w myworkspace
```

## MCP Tools Reference (29 tools)

### Tier 1: Instant Orientation

| Tool | Description |
|------|-------------|
| `codebase_map()` | Full project overview — call first every session |
| `search_symbols(query)` | Search across symbol names, code, and docs |
| `get_symbol(name)` | Full source code + metadata for a symbol |
| `get_signature(name)` | Just the signature (lightweight) |
| `symbols_in_file(path)` | Table of contents for a file |
| `list_projects()` | All projects in workspace with stats |

### Tier 2: Relationship Graph

| Tool | Description |
|------|-------------|
| `get_callers(name)` | Who calls this symbol? |
| `get_callees(name)` | What does this symbol call? |
| `get_dependents(name, transitive)` | Blast radius — what breaks if I change this? |
| `get_implementors(interface)` | All classes implementing an interface |
| `get_tests_for(name)` | Test functions covering a symbol |
| `get_type_hierarchy(name)` | Inheritance tree (base + subclasses) |

### Tier 3: Git Change Intelligence

| Tool | Description |
|------|-------------|
| `blame_symbol(name)` | Who changed this, when, and why |
| `recent_changes(n)` | Commit feed (cross-project in workspace) |
| `git_hotspots(n, since)` | Most frequently changed files |
| `whats_changed()` | Uncommitted work in progress |
| `changes_to(name)` | Commit history for a symbol's file |

### Tier 4: Build & Config

| Tool | Description |
|------|-------------|
| `get_build_targets()` | CMake/.csproj/npm targets with dependencies |
| `get_platform_variants(name)` | `#ifdef` platform guards around a symbol |
| `platform_conditionals()` | All platform-conditional code blocks |

### Tier 5: Semantic Search

| Tool | Description |
|------|-------------|
| `semantic_search(query)` | Find code by meaning (natural language) |
| `hybrid_search(query)` | Best of both: keyword + semantic with RRF |
| `embedding_status()` | Embedding coverage and model info |

### Tier 6: Meta & Server

| Tool | Description |
|------|-------------|
| `index_status()` | Index freshness and stats |
| `reindex()` | Trigger incremental re-index |
| `embedding_health()` | Check if embedding provider is reachable |
| `setup_guide()` | Structured setup instructions |
| `server_stats()` | Server uptime and process info |
| `restart_server()` | Request server restart (SSE only) |

### CLI Commands

| Command | Description |
|---------|-------------|
| `srclight index` | Index current directory (incremental by default) |
| `srclight search <query>` | Search the index |
| `srclight symbols <file>` | List symbols in a file |
| `srclight serve` | Start MCP server (stdio) |
| `srclight serve --workspace <name>` | Start MCP server (workspace SSE on :8742) |
| `srclight hook install` | Install git post-commit/post-checkout hooks |
| `srclight workspace init <name>` | Create a multi-repo workspace |
| `srclight workspace add <path> -w <name>` | Add a repo to a workspace |
| `srclight workspace index -w <name>` | Index all repos in a workspace |
| `srclight workspace status -w <name>` | Show workspace index status |

## Performance Notes

| Metric | Value |
|--------|-------|
| Index speed | ~1000 files/minute (CPU) |
| Search latency | <100ms (keyword); +embedding latency for semantic |
| Semantic query | ~3ms (GPU-resident cache via cupy) |
| Index size | ~1KB/file average |

- Incremental indexing only re-processes changed files (content hash detection)
- FTS5 keyword search is sub-millisecond
- GPU (cupy) optional — significant speedup for large embedding caches

## Integration with Serena

Srclight and Serena cover complementary ground:

| Query Type | Handler | Example |
|------------|---------|---------|
| Symbol lookup by name | Serena | "Find function `login_handler`" |
| Natural language concept | Srclight | "Where is auth handled?" |
| Caller/callee graph | Srclight | "What calls `validate_token`?" |
| Blast radius | Srclight | "What breaks if I rename this?" |
| Go to definition | Serena | "Jump to `UserService`" |
| Git blame | Srclight | "Who changed `auth_middleware`?" |

## Troubleshooting

### "No index found"

```bash
srclight index
```

### "Embedding server connection refused"

```bash
# Verify the internal server is reachable
curl http://inference.internal/v1/models
```

Re-index specifying the correct server:

```bash
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal srclight index \
  --embed openai:qwen3-embedding-8b
```

### "Semantic search returns no results"

Embeddings may not have been generated at index time. Check coverage:

```bash
# Via MCP tool
embedding_status()
```

Re-index with `--embed` if needed.

### "Slow indexing"

- Exclude large directories: `srclight index --exclude node_modules --exclude build`
- Check network latency to the internal embedding server

## Next Steps

- Read the [Architecture Overview](../architecture/overview.md)
- Set up [OpenCode](./opencode-quickstart.md) for orchestration
- Configure [Serena](./serena-quickstart.md) for symbolic navigation
