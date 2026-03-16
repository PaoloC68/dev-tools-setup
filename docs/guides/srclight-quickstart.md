# Srclight Quickstart Guide

## What is Srclight?

**Srclight** is the **semantic layer** of the Oh-My-OpenCode-Slim stack. It provides air-gap certified code indexing using local vector embeddings for natural language search across your codebase.

### Key Features
- **100% Offline** — All embeddings stored locally, no cloud dependency
- **Natural Language Search** — Find code by describing what you want
- **Similar Code Search** — Find patterns similar to a code snippet
- **SQLite Backend** — Lightweight, fast, embedded vector storage
- **Token Efficient** — Complements Serena's symbolic search with semantic understanding

## Installation

### Quick Install
```bash
pip install srclight

# With GPU support
pip install srclight[gpu]
```

### Pre-download Embedding Models (Air-gapped)
```bash
python -c "from sentence_transformers import SentenceTransformer; model = SentenceTransformer('all-MiniLM-L6-v2')"
```

### Install Language Servers (for code parsing)
```bash
# C/C++ - choose one
apt-get install clangd  # Recommended
# or
apt-get install ccls

# Python
pip install python-lsp-server

# TypeScript
npm install -g typescript-language-server
```

## Configuration

### Basic Configuration (.srclight/config.yml)
```yaml
network:
  offline_mode: true  # Mandatory for air-gap

parsers:
  - cpp
  - python
  - rust
  - typescript
  - javascript

database:
  path: .srclight/index.db

embeddings:
  model: all-MiniLM-L6-v2
  device: cpu  # or cuda for GPU
  cache: .srclight/models/
  dimensions: 384

indexing:
  batch_size: 100
  max_file_size: 1048576  # 1MB
  exclude:
    - "*.min.js"
    - "*.bundle.js"
    - "__pycache__/"
    - "node_modules/"
    - ".git/"
```

## Quick Start Workflow

### Step 1: Initialize Index
```bash
# Index entire project
srclight index /path/to/project

# Or with custom config
srclight index /path/to/project --config .srclight/config.yml
```

### Step 2: Incremental Updates
```bash
# After code changes
srclight update --incremental

# Full reindex
srclight index --rebuild
```

### Step 3: Search
```bash
# Natural language search
srclight search "authentication flow"

# Similar code search
srclight find-similar "function parseJson(input: string)"

# Filter by language
srclight search "JWT validation" --lang python
```

## Use Cases

### Use Case 1: Finding Code by Description

**Scenario**: You know what you want but not where it is

```
You: Where do we handle JWT token validation?

Srclight:
Found 5 matches:

1. src/auth/jwt.py:22 - validate_token()
   Score: 0.92
   Context: Validates JWT token signature and expiration

2. src/middleware/auth.py:15 - auth_middleware()
   Score: 0.87
   Context: Extracts and validates Bearer token

3. src/services/session.py:45 - check_session_valid()
   Score: 0.78
   Context: Validates session against token store
```

### Use Case 2: Finding Similar Code

**Scenario**: You have a code snippet and want to find similar patterns

```
You: Find all code similar to this function signature

Srclight:
Found 12 similar functions:

1. src/api/handlers/user.py:18 - create_user()
2. src/api/handlers/auth.py:22 - create_token()
3. src/services/cache.py:31 - create_entry()
...

Pattern: "create_*" functions returning result objects
```

### Use Case 3: Semantic Exploration

**Scenario**: Understand "how" something works across the codebase

```
You: How does our caching work?

Srclight:
Caching-related code (12 files):

Implementation patterns:
- Redis cache: src/cache/redis.py
- Memory cache: src/cache/memory.py
- Cache decorator: src/utils/cache.py

Usage patterns:
- @cached decorator for function results
- Manual cache.get()/set() calls
```

### Use Case 4: Bug Investigation

**Scenario**: Find all error handling related to a specific area

```
You: Find all error handling around database connections

Srclight:
Found 8 relevant locations:

1. src/db/connection.py:34
   try-except with retry logic

2. src/db/pool.py:52
   ConnectionPool timeout handling

3. src/api/middleware.py:28
   Global exception handler for DB errors
```

## Tools Reference

### Command Line Interface

| Command | Description |
|---------|-------------|
| `srclight index <path>` | Index a project directory |
| `srclight update` | Incremental index update |
| `srclight search <query>` | Natural language search |
| `srclight find-similar <code>` | Find similar code |
| `srclight stats` | Show index statistics |
| `srclight clear` | Clear the index |

### Python API

```python
from srclight import SemanticSearch

# Initialize
search = SemanticSearch(
    project_path="/path/to/project",
    model_name="all-MiniLM-L6-v2"
)

# Index
search.index()

# Search
results = search.semantic_search(
    query="authentication JWT token validation",
    top_k=5
)

# Similar
similar = search.find_similar(
    code_snippet="def validate_token(token):",
    top_k=3
)
```

### MCP Server Integration

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

## Performance Notes

| Metric | Value |
|--------|-------|
| Index speed | ~1000 files/minute |
| Search latency | <100ms (CPU) |
| Index size | ~1KB/file average |
| Embedding model | all-MiniLM-L6-v2 (384 dims) |

### Optimization Tips
- Use GPU for 10x faster indexing
- Exclude test files for smaller index
- Use incremental updates for active development
- Batch size of 100 optimal for most systems

## Integration with Serena

Srclight works alongside Serena for complete codebase understanding:

| Query Type | Handler | Example |
|------------|---------|----------|
| Symbol lookup | Serena | "Find function login_handler" |
| Natural language | Srclight | "Where is auth handled?" |
| File search | Both | "Find password validation" |

### Combined Workflow
```
You: Find where we validate user passwords

Srclight: [Semantic search]
Found in src/auth/validators.py:15

You: Show me that function and all its references

Serena: [Symbol search + navigation]
Function: validate_password()
References: 8 locations across 4 files
```

## Troubleshooting

### "Model not found"
```bash
# Re-download embedding model
python -c "from sentence_transformers import SentenceTransformer; model = SentenceTransformer('all-MiniLM-L6-v2')"
```

### "Index not found"
```bash
# Create new index
srclight index /path/to/project
```

### "Slow indexing"
- Check GPU is available: `nvidia-smi`
- Reduce batch_size in config
- Exclude more directories

### "Memory issues"
- Reduce batch_size
- Use device: cpu instead of cuda
- Exclude large files (binaries, builds)

## Next Steps

- Read the [Architecture Overview](../architecture/overview.md)
- Set up [OpenCode](./opencode-quickstart.md) for orchestration
- Configure [Serena](./serena-quickstart.md) for symbolic navigation
