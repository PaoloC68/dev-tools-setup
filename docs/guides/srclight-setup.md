# Srclight Setup Guide

Srclight is the **semantic layer** of the air-gapped AI coding architecture. It combines SQLite FTS5 keyword search, tree-sitter symbol extraction, and embeddings via an internal OpenAI-compatible server into a hybrid search engine for code.

## Overview

Srclight indexes your codebase using three complementary strategies: tree-sitter for structural symbols, FTS5 trigram + porter stemmer for keyword matching, and semantic embeddings for natural language queries. Results merge through Reciprocal Rank Fusion (RRF). It exposes 25 MCP tools and supports 7 languages (C, C++, Python, TypeScript, JavaScript, Rust, Go).

## Installation

```bash
pip install srclight
```

Embeddings are provided by the internal inference server. No local model installation is required — Srclight calls the server's OpenAI-compatible `/v1/embeddings` endpoint at index time and query time.

## Configuration

Srclight auto-generates config on first run. To customize, edit `.srclight/config.yml`:

```yaml
database:
  path: .srclight/index.db

embeddings:
  provider: openai-compatible
  base_url: http://inference.internal/v1
  model: text-embedding-gte-multilingual-base

indexing:
  batch_size: 100
  max_file_size: 1048576        # 1MB
  exclude:
    - "*.min.js"
    - "node_modules/"
    - "__pycache__/"
    - ".git/"
```

Replace `http://inference.internal/v1` with the actual base URL of your internal inference server.

## Local Fallback: infinity-emb

If the internal inference server is unavailable, `infinity-emb` provides a drop-in local replacement.
It serves the same model over the same OpenAI-compatible API — only the `base_url` changes.

```bash
# Install
pip install "infinity-emb[all]"

# Pre-download model (do this before going air-gapped)
python -c "from huggingface_hub import snapshot_download; snapshot_download('Alibaba-NLP/gte-multilingual-base')"

# Run (exposes http://localhost:7997/v1/embeddings)
infinity_emb v2 --model-name-or-path Alibaba-NLP/gte-multilingual-base --port 7997
```

Update `.srclight/config.yml` to point at the local server:

```yaml
embeddings:
  provider: openai-compatible
  base_url: http://localhost:7997/v1
  model: text-embedding-gte-multilingual-base
```

Set `HF_HUB_OFFLINE=1` after model download to prevent any Hugging Face network access:

```bash
export HF_HUB_OFFLINE=1
```

## Usage

### Indexing

```bash
# Index a project
srclight index /path/to/repo

# Named workspace (for multi-repo setups)
srclight index /path/to/repo --workspace myproject

# Incremental update after code changes
srclight update --incremental

# Full rebuild
srclight index --rebuild
```

### Searching

```bash
# Hybrid search (keyword + semantic via RRF)
srclight search "authentication flow"

# Filter by language
srclight search "JWT validation" --lang python

# Symbol-specific
srclight search "parseJson" --type function
```

### Multi-repo Workspaces

Srclight supports ATTACH across multiple SQLite databases and UNION across schemas:

```bash
srclight index /path/to/frontend --workspace frontend
srclight index /path/to/backend --workspace backend
srclight search "user auth" --workspace frontend,backend
```

## Integration with OpenCode

Start Srclight as an MCP server to expose all 25 tools to the orchestration layer:

```bash
srclight serve --workspace myproject
```

The 25 tools cover symbol search, relationship graphs (call graphs, type hierarchies), git change intelligence, semantic search, build system awareness, and document extraction. Serena handles structural/symbolic queries while Srclight handles semantic and hybrid queries.

## Performance Notes

| Metric | Value |
|--------|-------|
| Index speed | ~1000 files/minute (CPU) |
| Search latency | <100ms (hybrid) |
| Embedding latency | ~50-200ms per MCP call |
| Index size | ~1KB/file average |

- Incremental updates only re-index changed files
- FTS5 keyword search alone is sub-millisecond; embedding lookup adds network latency
- Reduce `batch_size` if the embedding server is under load

## Next Steps

- See the [Srclight Quickstart](./srclight-quickstart.md) for detailed use cases and tool reference
- Read the [Architecture Overview](../architecture/overview.md)
- Configure [Serena](./serena-quickstart.md) for symbolic navigation
