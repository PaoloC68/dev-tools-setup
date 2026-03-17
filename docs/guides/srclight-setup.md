# Srclight Setup Guide

Srclight is the **semantic layer** of the air-gapped AI coding architecture. It combines SQLite FTS5 keyword search, tree-sitter symbol extraction, and Ollama-powered embeddings into a hybrid search engine for code.

## Overview

Srclight indexes your codebase using three complementary strategies: tree-sitter for structural symbols, FTS5 trigram + porter stemmer for keyword matching, and semantic embeddings for natural language queries. Results merge through Reciprocal Rank Fusion (RRF). It exposes 25 MCP tools and supports 7 languages (C, C++, Python, TypeScript, JavaScript, Rust, Go).

## Installation

```bash
pip install srclight
```

Ollama is required for local embeddings:

```bash
# macOS
brew install ollama

# Linux
curl -fsSL https://ollama.com/install.sh | sh

# Pull the default embedding model (~6GB VRAM)
ollama pull qwen3-embedding

# Or the lighter alternative (~4GB VRAM)
ollama pull nomic-embed-text
```

For air-gapped setups, pull the Ollama model **before** disconnecting from the network. Once pulled, Ollama serves models entirely offline.

## Configuration

Srclight auto-generates config on first run. To customize, edit `.srclight/config.yml`:

```yaml
database:
  path: .srclight/index.db

embeddings:
  provider: ollama
  model: qwen3-embedding       # Default: best local quality
  # model: nomic-embed-text    # Alternative: lighter, 8GB VRAM

indexing:
  batch_size: 100
  max_file_size: 1048576        # 1MB
  exclude:
    - "*.min.js"
    - "node_modules/"
    - "__pycache__/"
    - ".git/"
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

- Ollama with GPU gives 5-10x faster embedding generation
- Incremental updates only re-index changed files
- FTS5 keyword search alone is sub-millisecond
- Switch to `nomic-embed-text` on memory-constrained systems

## Next Steps

- See the [Srclight Quickstart](./srclight-quickstart.md) for detailed use cases and tool reference
- Read the [Architecture Overview](../architecture/overview.md)
- Configure [Serena](./serena-quickstart.md) for symbolic navigation
