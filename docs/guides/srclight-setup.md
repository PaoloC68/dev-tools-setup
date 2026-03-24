# Srclight Setup Guide

Srclight is the **semantic layer** of the air-gapped AI coding architecture. It combines SQLite FTS5
keyword search, tree-sitter symbol extraction, and embeddings via an internal OpenAI-compatible
server into a hybrid search engine for code.

## Overview

Srclight indexes your codebase using four complementary strategies: tree-sitter for structural
symbols, three FTS5 indexes for keyword matching (names, trigram, docstrings), and optional semantic
embeddings for natural language queries. Results merge through Reciprocal Rank Fusion (RRF). It
exposes 29 MCP tools and supports 11 languages.

## Installation

> **Version requirement**: OpenAI-compatible embedding endpoints require v0.11.0+. The current
> release is v0.15.1. Versions ≤ 0.8.1 only support Ollama and Voyage — `pip install --upgrade`
> stays at 0.8.1 in an air-gapped environment if the internal mirror hasn't been updated.

> **Known issue**: `tree-sitter-dart` is declared as a dependency of srclight 0.15.1 but is
> not published on PyPI. Install with `--no-deps` and add all other dependencies manually.
> The pip resolver will warn about the missing package — this is harmless. Dart indexing is
> simply unavailable, which is irrelevant for C/C++ codebases.

### Connected Machine (pre-air-gap)

```bash
# Download srclight wheel only (tree-sitter-dart is not on PyPI)
pip download "srclight==0.15.1" --no-deps -d ./srclight-wheels/

# Download all required dependencies except tree-sitter-dart
pip download \
  click mcp numpy \
  "tree-sitter>=0.21" \
  tree-sitter-c tree-sitter-cpp tree-sitter-c-sharp \
  tree-sitter-python tree-sitter-javascript tree-sitter-typescript \
  tree-sitter-go tree-sitter-java tree-sitter-kotlin \
  tree-sitter-rust tree-sitter-swift tree-sitter-php \
  tree-sitter-markdown \
  -d ./srclight-wheels/

# Transfer ./srclight-wheels/ to the air-gapped machine
```

### Air-gapped Machine

```bash
# Install srclight (no-deps to bypass tree-sitter-dart)
pip install --no-index --find-links ./srclight-wheels/ --no-deps srclight==0.15.1

# Install all actual dependencies
pip install --no-index --find-links ./srclight-wheels/ \
  click mcp numpy \
  tree-sitter \
  tree-sitter-c tree-sitter-cpp tree-sitter-c-sharp \
  tree-sitter-python tree-sitter-javascript tree-sitter-typescript \
  tree-sitter-go tree-sitter-java tree-sitter-kotlin \
  tree-sitter-rust tree-sitter-swift tree-sitter-php \
  tree-sitter-markdown

# Verify (pip will warn about tree-sitter-dart — expected and harmless)
pip show srclight    # must show Version: 0.15.1
srclight --version   # must print srclight, version 0.15.1
```

Embeddings are provided by the internal inference server. No local model installation is required —
Srclight calls the server's OpenAI-compatible `/v1/embeddings` endpoint at index time and query time.

## Configuration

**Srclight has no config file.** All configuration is via CLI flags at index time. The `.srclight/`
directory contains only `index.db` (SQLite) and optional `.npy` embedding sidecars.

> `srclight index` automatically adds `.srclight/` to your `.gitignore`.

## Indexing

```bash
# Keyword-only index (no embeddings)
cd /path/to/repo
srclight index

# Index with embeddings via internal OpenAI-compatible server
# Pass the server URL to --embed and the model name to --embed-model:
OPENAI_API_KEY=sk-xxx srclight index \
  --embed http://inference.internal/v1 \
  --embed-model qwen3-embedding-8b
```

Replace `http://inference.internal/v1` with the actual base URL of your internal inference server.

> **Provider selection**: When `--embed` receives a URL (starting with `http`), Srclight uses
> the OpenAI-compatible provider regardless of the model name. Any model served by the internal
> server works — pass its exact name via `--embed-model`.

Indexing is incremental by default — only re-indexes files whose content hash changed. Re-run
`srclight index` at any time; it will only process what has changed.

## Local Fallback: infinity-emb

If the internal inference server is unavailable, `infinity-emb` provides a drop-in local replacement
serving the same model over the same OpenAI-compatible API.

```bash
# Install
pip install "infinity-emb[all]"

# Pre-download model before going air-gapped
python -c "from huggingface_hub import snapshot_download; snapshot_download('Qwen/Qwen3-Embedding-8B')"
export HF_HUB_OFFLINE=1

# Run (exposes http://localhost:7997/v1/embeddings)
infinity_emb v2 --model-name-or-path Qwen/Qwen3-Embedding-8B --port 7997
```

Index using the local server:

```bash
srclight index --embed http://localhost:7997/v1 --embed-model qwen3-embedding-8b
# No OPENAI_API_KEY needed for local infinity-emb
```

## Usage

### Single Repo

```bash
# Index
srclight index
OPENAI_API_KEY=sk-xxx srclight index --embed http://inference.internal/v1 --embed-model qwen3-embedding-8b

# Search
srclight search "authentication flow"
srclight search --kind function "parseJson"

# Start MCP server (stdio)
srclight serve

# Register with Claude Code
claude mcp add srclight -- srclight serve
```

### Multi-repo Workspaces

```bash
# Create workspace
srclight workspace init myworkspace

# Add repos
srclight workspace add /path/to/repo1 -w myworkspace
srclight workspace add /path/to/repo2 -w myworkspace

# Index all repos
OPENAI_API_KEY=sk-xxx srclight workspace index -w myworkspace \
  --embed http://inference.internal/v1 --embed-model qwen3-embedding-8b

# Start MCP server (SSE — persistent, port 8742)
srclight serve --workspace myworkspace &
claude mcp add --transport sse srclight http://127.0.0.1:8742/sse

# Status
srclight workspace status -w myworkspace
```

### Auto-reindex (Git Hooks)

```bash
# Install hooks in current repo
srclight hook install

# Install across all repos in a workspace
srclight hook install --workspace myworkspace
```

## Integration with OpenCode

Add Srclight to `opencode.json` as an MCP server:

```json
{
  "mcp": {
    "srclight": {
      "type": "local",
      "command": ["srclight", "serve"],
      "enabled": true
    }
  }
}
```

For workspace mode (SSE):

```json
{
  "mcp": {
    "srclight": {
      "type": "remote",
      "url": "http://127.0.0.1:8742/sse",
      "enabled": true
    }
  }
}
```

The 29 MCP tools cover symbol search, relationship graphs (callers, callees, blast radius),
git change intelligence, semantic search, build system awareness, and document extraction.
Serena handles structural/symbolic queries; Srclight handles semantic and hybrid queries.

## Performance Notes

| Metric | Value |
|--------|-------|
| Index speed | ~1000 files/minute (CPU) |
| Search latency | <100ms (keyword); ~3ms (semantic, GPU cache) |
| Index size | ~1KB/file average |

- Incremental indexing only re-processes changed files (content hash detection)
- FTS5 keyword search is sub-millisecond
- GPU (cupy) optional: `pip install 'srclight[gpu]'` for CUDA-accelerated semantic search

## Next Steps

- See the [Srclight Quickstart](./srclight-quickstart.md) for detailed use cases and tool reference
- Read the [Architecture Overview](../architecture/overview.md)
- Configure [Serena](./serena-quickstart.md) for symbolic navigation
