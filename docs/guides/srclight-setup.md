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

```bash
pip install srclight
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

# Index with embeddings via internal server
# Model names starting with "text-embedding" are auto-detected as OpenAI-compatible:
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal srclight index \
  --embed openai:text-embedding-gte-multilingual-base

# For other model names, use the "openai:" prefix to force OpenAI-compatible provider:
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal srclight index \
  --embed openai:qwen3-embedding-8b
```

Replace `http://inference.internal` with the actual base URL of your internal inference server.
`OPENAI_BASE_URL` must NOT include `/v1` — Srclight appends `/v1/embeddings` automatically.

**Provider auto-detection** (from Srclight source — `embeddings.py`):

| Model name | Provider selected |
|------------|-------------------|
| starts with `text-embedding` | OpenAI-compatible |
| starts with `voyage` | Voyage AI |
| starts with `embed-v3` or `embed-v4` | Cohere |
| `openai:<model>` prefix | OpenAI-compatible (explicit) |
| anything else | Ollama ← **will fail if Ollama not running** |

Using `openai:` prefix explicitly (as shown above) is safer than relying on auto-detection —
it works regardless of the model name and makes the intent unambiguous.

Indexing is incremental by default — only re-indexes files whose content hash changed. Re-run
`srclight index` at any time; it will only process what has changed.

## Local Fallback: infinity-emb

If the internal inference server is unavailable, `infinity-emb` provides a drop-in local replacement
serving the same model over the same OpenAI-compatible API.

```bash
# Install
pip install "infinity-emb[all]"

# Pre-download model before going air-gapped
python -c "from huggingface_hub import snapshot_download; snapshot_download('Alibaba-NLP/gte-multilingual-base')"
export HF_HUB_OFFLINE=1

# Run (exposes http://localhost:7997/v1/embeddings)
infinity_emb v2 --model-name-or-path Alibaba-NLP/gte-multilingual-base --port 7997
```

Index using the local server:

```bash
OPENAI_BASE_URL=http://localhost:7997 srclight index --embed openai:text-embedding-gte-multilingual-base
# No OPENAI_API_KEY needed for local infinity-emb
```

## Usage

### Single Repo

```bash
# Index
srclight index
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal srclight index --embed openai:text-embedding-gte-multilingual-base

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
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal \
  srclight workspace index -w myworkspace --embed openai:text-embedding-gte-multilingual-base

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
