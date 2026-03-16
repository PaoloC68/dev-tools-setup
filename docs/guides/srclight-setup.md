# Srclight Setup Guide

Srclight is the **semantic layer** of the Oh-My-OpenCode-Slim stack, providing air-gap certified code indexing and vector database search capabilities.

## Overview

Srclight enables natural language code search using local embeddings, making it ideal for offline/air-gapped development environments.

## Installation

```bash
pip install srclight[gpu]
```

For air-gapped setups, pre-download embedding models:

```bash
python -c "from sentence_transformers import SentenceTransformer; model = SentenceTransformer('all-MiniLM-L6-v2')"
```

## Configuration

Create `.srclight/config.yml` at your project root:

```yaml
network:
  offline_mode: true  # Mandatory for air-gap

parsers:
  - cpp
  - python
  - rust

database:
  path: .srclight/index.db


embeddings:
  model: all-MiniLM-L6-v2
  device: cpu  # or cuda for GPU acceleration
  cache: .srclight/models/
```

## Usage

### Indexing a Repository

```bash
# Initial index build
srclight index /path/to/repo

# Incremental updates
srclight update --incremental
```

### Semantic Search

Srclight provides two primary search capabilities:

1. **Natural Language Search**: Find code by describing what you want
   ```
   semantic_search("Find JSON parsing logic")
   ```

2. **Similar Code Search**: Find patterns similar to a code snippet
   ```
   find_similar(snippet="void parseJson(const std::string& input)")
   ```

## Integration with OpenCode


Srclight operates as an MCP server, integrated into the OpenCode orchestration layer. It handles semantic queries while Serena handles structural/symbolic queries.

## Performance Notes

- First-time indexing scales with repository size
- Incremental updates are recommended for active development
- CPU vs GPU affects embedding speed, not accuracy
