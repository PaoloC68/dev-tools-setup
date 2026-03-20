# Memora Quickstart Guide

## What is Memora?

**Memora** (github.com/agentic-box/memora) is the **persistence layer** of the air-gapped AI coding
architecture. A lightweight MCP server that gives AI agents cross-session memory via SQLite, semantic
search, and an interactive knowledge graph.

Without Memora, every session starts from zero. Architectural decisions, discovered patterns, and
accumulated project context vanish when the context window resets. Memora persists that knowledge
locally and makes it searchable across sessions.

### Key Features

- **Cross-Session Memory**: Decisions and context survive session resets
- **Hybrid Search**: Full-text (FTS) + semantic vector search via Reciprocal Rank Fusion
- **Knowledge Graph**: Interactive visualization at `localhost:8765/graph`
- **100% Offline**: Local SQLite + local embeddings via `sentence-transformers`
- **Hierarchical Organization**: Section/subsection structure with tag filtering
- **Memory Linking**: Typed edges between memories (implements, supersedes, references, etc.)

## Installation

### Base Install

```bash
pip install git+https://github.com/agentic-box/memora.git
```

Includes cloud storage (S3/R2) and OpenAI embeddings. For air-gapped deployments, install the local
extras instead:

### Air-Gapped Install (Required)

```bash
# Includes sentence-transformers for fully offline embeddings (~2GB for PyTorch)
pip install "memora[local] @ git+https://github.com/agentic-box/memora.git"
```

> **Air-Gap Warning**: The base install defaults to OpenAI embeddings, which require a network call.
> Always use `memora[local]` for air-gapped environments.

### Air-Gapped Preparation Checklist

```bash
# 1. Install with local extras while online
pip install "memora[local] @ git+https://github.com/agentic-box/memora.git"

# 2. Pre-download the sentence-transformers model
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"

# 3. Verify the server starts
memora-server --help

# 4. Disconnect from network. Everything runs offline from here.
```

## Configuration

### Claude Code / OpenCode

Add to `.mcp.json` in your project root:

```json
{
  "mcpServers": {
    "memora": {
      "command": "memora-server",
      "args": [],
      "env": {
        "MEMORA_DB_PATH": "~/.local/share/memora/memories.db",
        "MEMORA_EMBEDDING_MODEL": "sentence-transformers",
        "MEMORA_ALLOW_ANY_TAG": "1",
        "MEMORA_GRAPH_PORT": "8765"
      }
    }
  }
}
```

> **Note**: `MEMORA_EMBEDDING_MODEL=sentence-transformers` is required for air-gapped operation.
> The default (`openai`) makes network calls to the OpenAI API.

### Environment Variables Reference

| Variable | Description | Air-Gap Value |
|----------|-------------|---------------|
| `MEMORA_DB_PATH` | SQLite database path | `~/.local/share/memora/memories.db` |
| `MEMORA_EMBEDDING_MODEL` | Embedding backend: `openai`, `sentence-transformers`, `tfidf` | `sentence-transformers` |
| `SENTENCE_TRANSFORMERS_MODEL` | Model for sentence-transformers | `all-MiniLM-L6-v2` (default) |
| `MEMORA_ALLOW_ANY_TAG` | Allow any tag without allowlist validation | `1` |
| `MEMORA_TAG_FILE` | Path to file with allowed tags (one per line) | — |
| `MEMORA_GRAPH_PORT` | Port for knowledge graph visualization server | `8765` |
| `MEMORA_LLM_ENABLED` | Enable LLM-powered deduplication | `false` (air-gap) |
| `MEMORA_STORAGE_URI` | Cloud storage URI — **MUST NOT be set** for air-gap | unset |
| `CLOUDFLARE_API_TOKEN` | Cloudflare D1 access — **MUST NOT be set** for air-gap | unset |

### Security Configuration

For air-gapped deployments, verify no cloud variables are set:

```bash
# These MUST NOT be set:
env | grep -i memora
env | grep -i cloudflare

# Unset if present:
unset MEMORA_STORAGE_URI
unset CLOUDFLARE_API_TOKEN
unset MEMORA_CLOUD_ENCRYPT
unset MEMORA_CLOUD_COMPRESS
```

## Quick Start Workflow

### Step 1: Start the MCP Server

The server starts automatically when Claude Code / OpenCode launches with the `.mcp.json` config above.

Manual invocation:

```bash
# Default: stdio mode for MCP
memora-server

# With knowledge graph visualization
memora-server --graph-port 8765

# Disable graph server (headless environments)
memora-server --no-graph
```

### Step 2: Store a Memory

```
You: Remember this architectural decision: all database access goes through the repository layer.
     Direct ORM calls from handlers are forbidden.

Memora:
  Stored memory #1
  Tags: architecture, database
  Section: decisions
```

### Step 3: Search Across Sessions

```
You: What did we decide about database access?

Memora (hybrid search):
  Found 2 matches:

  #1 — Architecture Decision (score: 0.94)
  "All database access goes through the repository layer.
   Direct ORM calls from handlers are forbidden."
  Tags: architecture, database | Created: 2026-03-15

  #3 — Repository Pattern (score: 0.71)
  "UserRepository wraps all User model queries.
   See src/repositories/user.py"
  Tags: architecture, patterns | Created: 2026-03-14
```

### Step 4: View the Knowledge Graph

```bash
# Open in browser after starting memora-server
open http://localhost:8765/graph
```

The graph shows memories as nodes, cross-references as edges, and supports tag/section filtering.

## MCP Tools Reference

### Core Memory Tools

| Tool | Description |
|------|-------------|
| `memory_create` | Store a new memory with content, tags, and section |
| `memory_search` | Hybrid search (FTS + semantic) across all memories |
| `memory_list` | List memories with tag/section/date filters |
| `memory_update` | Update memory content or metadata |
| `memory_delete` | Remove a specific memory |
| `memory_get` | Retrieve a memory by ID |

### Structured Memory Tools

| Tool | Description |
|------|-------------|
| `memory_create_todo` | Create a TODO with status and priority |
| `memory_create_issue` | Create an issue with severity and component |
| `memory_create_section` | Create a section placeholder for organization |
| `memory_create_batch` | Store multiple memories in one call |

### Knowledge Graph Tools

| Tool | Description |
|------|-------------|
| `memory_link` | Create typed edge between two memories |
| `memory_unlink` | Remove a link between memories |
| `memory_boost` | Increase a memory's importance for ranking |
| `memory_clusters` | Detect clusters of related memories |

### Maintenance Tools

| Tool | Description |
|------|-------------|
| `memory_find_duplicates` | Find similar memories for deduplication |
| `memory_merge` | Merge two memories (append/prepend/replace) |
| `memory_rebuild_embeddings` | Rebuild all embeddings after model change |
| `memory_rebuild_crossrefs` | Rebuild cross-reference graph |
| `memory_insights` | Activity summary, stale detection, consolidation suggestions |
| `memory_export_graph` | Export knowledge graph as static HTML |

## Use Cases

### Use Case 1: Persisting Architectural Decisions

**Scenario**: Capture a decision so future sessions don't re-debate it.

```
You: Save this: We chose SQLite over PostgreSQL because the deployment target is a single-node
     air-gapped workstation. No connection pooling needed.

Memora:
  Stored memory #12
  Tags: architecture, database, decisions
  Section: decisions/infrastructure

[Next session, 3 days later]

You: Why are we using SQLite?

Memora:
  #12 — Infrastructure Decision (score: 0.97)
  "Chose SQLite over PostgreSQL: deployment target is single-node air-gapped workstation.
   No connection pooling needed."
```

### Use Case 2: Tracking Open Issues

**Scenario**: Log a bug found during exploration so it isn't forgotten.

```
You: Create an issue: The auth middleware doesn't handle expired refresh tokens.
     It returns 500 instead of 401. Severity: major. Component: auth.

Memora:
  Created issue #23
  Status: open | Severity: major | Component: auth
  Tags: bug, auth

[Later]

You: What open issues do we have in auth?

Memora:
  Found 2 open issues tagged 'auth':

  #23 — major: "Auth middleware returns 500 on expired refresh tokens (should be 401)"
  #19 — minor: "Login endpoint doesn't rate-limit failed attempts"
```

### Use Case 3: Cross-Session Context Recovery

**Scenario**: Resume work after a session reset without re-exploring the codebase.

```
You: What were we working on last session?

Memora (memory_insights, period="1d"):
  Activity summary (last 24h):
  - 8 memories created
  - Focus areas: auth refactor, JWT token handling
  - Open TODOs: 3 (2 in-progress)

  In-progress:
  #31 — TODO: "Refactor validate_token to raise AuthError instead of returning False"
  #28 — TODO: "Add refresh token rotation to login handler"
```

### Use Case 4: Linking Related Decisions

**Scenario**: Connect memories that reference each other.

```
You: Link memory #12 (SQLite decision) to memory #8 (repository pattern decision).
     Edge type: related_to.

Memora:
  Linked #12 → #8 (related_to, bidirectional)
  Knowledge graph updated.
```

## Embedding Backends

| Backend | Install | Quality | Offline | Notes |
|---------|---------|---------|---------|-------|
| `sentence-transformers` | `memora[local]` | Good | Yes | Recommended for air-gap |
| `tfidf` | Included | Basic | Yes | No model download needed |
| `openai` | Included | High | No | Requires `OPENAI_API_KEY` |

For air-gapped deployments, use `sentence-transformers`. The `tfidf` backend requires no model
download and works immediately but provides keyword-only matching without semantic understanding.

**Switching backends** requires rebuilding embeddings:

```bash
# After changing MEMORA_EMBEDDING_MODEL, rebuild:
memory_rebuild_embeddings
memory_rebuild_crossrefs
```

## Air-Gap Compliance Checklist

| Check | Required | Verification |
|-------|----------|-------------|
| `MEMORA_STORAGE_URI` not set | Yes | `env \| grep MEMORA_STORAGE_URI` → empty |
| `CLOUDFLARE_API_TOKEN` not set | Yes | `env \| grep CLOUDFLARE` → empty |
| `MEMORA_EMBEDDING_MODEL=sentence-transformers` | Yes | Check `.mcp.json` env block |
| `sentence-transformers` model pre-downloaded | Yes | `python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('all-MiniLM-L6-v2')"` |
| `MEMORA_LLM_ENABLED` not set or `false` | Yes | LLM dedup requires OpenAI API |
| `MEMORA_DB_PATH` points to local path | Yes | Verify path is local, not a URI |

## Troubleshooting

### "memora-server: command not found"

```bash
# Verify installation
pip show memora

# Check PATH includes pip bin directory
which memora-server

# Reinstall
pip install "memora[local] @ git+https://github.com/agentic-box/memora.git"
```

### "OpenAI API error" or network calls on startup

```bash
# Set embedding model to local backend
export MEMORA_EMBEDDING_MODEL=sentence-transformers

# Or add to .mcp.json env block:
# "MEMORA_EMBEDDING_MODEL": "sentence-transformers"
```

### "sentence_transformers not found"

```bash
# Install local extras
pip install "memora[local] @ git+https://github.com/agentic-box/memora.git"
```

### "Graph server not accessible"

```bash
# Verify server started with graph enabled (no --no-graph flag)
memora-server --graph-port 8765

# Check port is not in use
lsof -i :8765
```

### "Embeddings out of date after model change"

```bash
# Rebuild after switching MEMORA_EMBEDDING_MODEL
memory_rebuild_embeddings
memory_rebuild_crossrefs
```

## Integration with Serena and Srclight

Memora complements the other layers with a distinct role:

| Layer | Tool | What It Stores | Query Type |
|-------|------|----------------|------------|
| Symbolic | Serena | Code structure (symbols, references) | "Find function login_handler" |
| Semantic | Srclight | Code content (indexed files) | "Where is auth handled?" |
| Memory | Memora | Agent knowledge (decisions, context) | "What did we decide about tokens?" |

Serena's `write_memory` / `read_memory` tools write to `.serena/memories/` (markdown files).
Memora stores structured, searchable records in SQLite. Use both: Serena for project-specific
symbol-level notes, Memora for cross-project decisions and session context.

## Next Steps

- Read the [Architecture Overview](../architecture/overview.md)
- Set up [OpenCode](./opencode-quickstart.md) for orchestration
- Configure [Serena](./serena-quickstart.md) for symbolic navigation
- Configure [Srclight](./srclight-quickstart.md) for semantic code search
