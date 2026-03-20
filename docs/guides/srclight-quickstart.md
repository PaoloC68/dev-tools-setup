# Srclight Quickstart Guide

## What is Srclight?

**Srclight** is the **semantic layer** of the air-gapped AI coding architecture. Deep code indexing for AI agents, built on SQLite FTS5, tree-sitter, embeddings, and the Model Context Protocol (MCP).

### Core Architecture

Srclight combines three indexing strategies into a single hybrid search engine:

1. **Tree-sitter parsing** extracts precise symbols (functions, classes, methods, interfaces, structs), builds call graph edges, and maps type hierarchies across 7 supported languages.
2. **FTS5 trigram + porter stemmer** provides fast keyword search with fuzzy matching.
3. **Semantic embeddings** via an OpenAI-compatible embedding endpoint enable natural language queries against your codebase.

Search results merge keyword and semantic matches through **Reciprocal Rank Fusion (RRF)**, giving you the best of both approaches in a single query.

### Key Features

- **Hybrid Search** combines FTS5 keyword matching with semantic embeddings via RRF
- **100% Offline** when pointed at an internal embedding server
- **25 MCP Tools** covering symbol search, relationship graphs, git change intelligence, semantic search, build system awareness, and document extraction
- **Tree-sitter Symbol Extraction** for precise structural understanding
- **Multi-repo Workspaces** via ATTACH across SQLite databases and UNION across schemas
- **SQLite Backend** for lightweight, fast, embedded storage
- **7 Language Support** via tree-sitter grammars

### Supported Languages

| Language | Tree-sitter Grammar | Symbol Types |
|----------|---------------------|--------------|
| C | tree-sitter-c | functions, structs, enums, typedefs |
| C++ | tree-sitter-cpp | classes, methods, functions, structs, namespaces |
| Python | tree-sitter-python | classes, functions, methods, decorators |
| TypeScript | tree-sitter-typescript | classes, interfaces, functions, types |
| JavaScript | tree-sitter-javascript | classes, functions, methods |
| Rust | tree-sitter-rust | structs, enums, traits, functions, impls |
| Go | tree-sitter-go | structs, interfaces, functions, methods |

## Installation

### Quick Install

```bash
pip install srclight
```

### Air-gapped Preparation Checklist

```bash
# 1. Install Srclight
pip install srclight

# 2. Verify the internal embedding server is reachable
curl http://inference.internal/v1/models

# 3. Disconnect from external network. Everything runs via the internal server.
```

## Configuration

Srclight creates its configuration automatically on first run. The config lives in `.srclight/` at your project root.

### Configuration Reference (.srclight/config.yml)

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
    - "*.bundle.js"
    - "__pycache__/"
    - "node_modules/"
    - ".git/"
```

Replace `http://inference.internal/v1` with the actual base URL of your internal inference server.

### Embedding Model Options

| Model | Provider | Offline | Notes |
|-------|----------|---------|-------|
| `text-embedding-gte-multilingual-base` | Internal server | Yes | Default for this deployment |

## Quick Start Workflow

### Step 1: Index Your Project

```bash
# Index entire project
srclight index /path/to/project

# Index with a named workspace (for multi-repo setups)
srclight index /path/to/project --workspace myproject
```

Tree-sitter parses each file, extracts symbols and relationships, then the embedding server generates vectors. The resulting index lands in `.srclight/index.db`.

### Step 2: Start the MCP Server

```bash
# Serve as MCP server for OpenCode integration
srclight serve --workspace myproject
```

This exposes all 25 MCP tools to the orchestration layer.

### Step 3: Search

```bash
# Hybrid search (keyword + semantic, merged via RRF)
srclight search "authentication flow"

# Filter by language
srclight search "JWT validation" --lang python

# Symbol-specific search
srclight search "parseJson" --type function
```

### Step 4: Incremental Updates

```bash
# After code changes, update only modified files
srclight update --incremental

# Full reindex when needed
srclight index --rebuild
```

## Use Cases

### Use Case 1: Finding Code by Description

**Scenario**: You know what you want but not where it is.

```
You: Where do we handle JWT token validation?

Srclight (hybrid search):
Found 5 matches (keyword + semantic fusion):

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

**Scenario**: You have a code snippet and want to find similar patterns.

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

**Scenario**: Understand "how" something works across the codebase.

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

**Scenario**: Find all error handling related to a specific area.

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

### Use Case 5: Multi-repo Search

**Scenario**: Search across multiple repositories in a single query.

```bash
# Index multiple repos into separate workspaces
srclight index /path/to/frontend --workspace frontend
srclight index /path/to/backend --workspace backend
srclight index /path/to/shared-lib --workspace shared

# Srclight ATTACHes SQLite databases and UNIONs across schemas
srclight search "user authentication" --workspace frontend,backend,shared
```

## MCP Tools Reference

Srclight exposes 25 tools through the MCP protocol. Key categories:

### Symbol Search Tools

| Tool | Description |
|------|-------------|
| `search_symbols` | Find symbols by name pattern |
| `search_semantic` | Natural language code search |
| `search_hybrid` | Combined keyword + semantic search (RRF) |
| `find_similar` | Find code similar to a snippet |

### Relationship Graph Tools

| Tool | Description |
|------|-------------|
| `get_call_graph` | Function call relationships |
| `get_type_hierarchy` | Class/struct inheritance trees |
| `get_references` | Where a symbol is used |
| `get_dependencies` | Module/file dependency graph |

### Git Change Intelligence Tools

| Tool | Description |
|------|-------------|
| `get_recent_changes` | Recently modified symbols |
| `get_change_frequency` | Hot spots in the codebase |
| `get_co_change_patterns` | Files that change together |

### Build System and Document Tools

| Tool | Description |
|------|-------------|
| `get_build_targets` | Build system target analysis |
| `extract_docs` | Extract documentation from code |
| `get_file_summary` | Summarize file contents |

### CLI Commands

| Command | Description |
|---------|-------------|
| `srclight index <path>` | Index a project directory |
| `srclight update --incremental` | Incremental index update |
| `srclight search <query>` | Hybrid search (keyword + semantic) |
| `srclight serve --workspace <name>` | Start MCP server |
| `srclight stats` | Show index statistics |
| `srclight clear` | Clear the index |

## Performance Notes

| Metric | Value |
|--------|-------|
| Index speed | ~1000 files/minute (CPU) |
| Search latency | <100ms (hybrid search) |
| Index size | ~1KB/file average |
| Embedding latency | ~50-200ms per MCP call (network-dependent) |

### Optimization Tips

- Exclude test files and generated code for a smaller index
- Incremental updates for active development (only re-indexes changed files)
- FTS5 keyword search alone is sub-millisecond; embedding lookup adds the latency
- Multi-repo workspaces share a single embedding server endpoint

## Integration with Serena

Srclight works alongside Serena for complete codebase understanding:

| Query Type | Handler | Example |
|------------|---------|---------|
| Symbol lookup | Serena | "Find function login_handler" |
| Natural language | Srclight | "Where is auth handled?" |
| Relationship graph | Srclight | "What calls validate_token?" |
| Code navigation | Serena | "Go to definition of UserService" |
| Combined | Both | "Find password validation and show references" |

### Combined Workflow

```
You: Find where we validate user passwords

Srclight: [Hybrid search - keyword + semantic]
Found in src/auth/validators.py:15

You: Show me that function and all its references

Serena: [Symbol search + navigation]
Function: validate_password()
References: 8 locations across 4 files
```

## Troubleshooting

### "Embedding server connection refused"

```bash
# Verify the internal server is reachable
curl http://inference.internal/v1/models

# Check Srclight config points to the correct base_url
cat .srclight/config.yml
```

### "Model not found"

```bash
# List models available on the internal server
curl http://inference.internal/v1/models

# Verify the model name in .srclight/config.yml matches exactly
```

### "Index not found"

```bash
# Create new index
srclight index /path/to/project
```

### "Slow indexing"

- Reduce `batch_size` in config to lower concurrent embedding requests
- Exclude large directories (node_modules, build artifacts)
- Check network latency to the internal embedding server

### "Memory issues"

- Reduce `batch_size` in config
- Exclude large files (binaries, build outputs)

## Next Steps

- Read the [Architecture Overview](../architecture/overview.md)
- Set up [OpenCode](./opencode-quickstart.md) for orchestration
- Configure [Serena](./serena-quickstart.md) for symbolic navigation
