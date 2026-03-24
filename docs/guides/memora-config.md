# Memora Configuration Guide

Memora is the **persistence layer** of the air-gapped AI coding architecture, providing multi-session memory and context management for air-gapped development environments.

## Overview

Memora stores cross-session context, research findings, and architectural decisions using a local SQLite backend with hybrid keyword + semantic search.

> **Air-Gap Warning**: Memora includes optional cloud sync features (S3, R2, Cloudflare D1).
> These MUST remain disabled for air-gapped deployments. See [Security Configuration](#security-configuration) below.

> **Note**: For installation and first-time setup, see the [Memora Quickstart](./memora-quickstart.md).
> This document covers configuration reference only.

## Directory Structure

```
.serena/
├── memories/           # Serena's persistent insights (markdown/text)
└── project.yml        # Serena project configuration

~/.local/share/memora/
└── memories.db        # Memora SQLite database (default MEMORA_DB_PATH)
```

> **Note**: Memora does not create a `.memora/` project directory. The database lives at
> `~/.local/share/memora/memories.db` by default, configurable via `MEMORA_DB_PATH`.

## Configuration Options

### Serena Configuration (.serena/project.yml)

```yaml
# Core settings
read_only: false           # Enable/disable file modifications
enable_gui_logging: true   # Enable GUI-based logging
onboarding_enabled: true   # Enable automated project onboarding

# Language support
languages:
  - cpp
  - python
  - rust

# Exclusions (recommended for large codebases)
exclusions:
  - "**/node_modules/**"
  - "**/*build*"
  - "**/deps/**"
  - ".git"
```

### Memora MCP Configuration (.mcp.json)

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

> **Air-Gap Note**: `MEMORA_EMBEDDING_MODEL=sentence-transformers` is required. The default
> (`openai`) makes network calls to the OpenAI API.

### Security Configuration

For air-gapped deployments, ensure these environment variables are **NOT set**:

```bash
# These MUST NOT be set in air-gapped environments:
# MEMORA_STORAGE_URI        — Enables S3/R2/D1 cloud sync
# CLOUDFLARE_API_TOKEN      — Enables Cloudflare D1 access
# MEMORA_CLOUD_ENCRYPT      — Cloud encryption (implies cloud usage)
# MEMORA_CLOUD_COMPRESS     — Cloud compression (implies cloud usage)

# Verify no cloud sync is configured:
env | grep -i memora
env | grep -i cloudflare
```

Only set these safe variables:

```bash
# Safe for air-gapped:
export MEMORA_DB_PATH="$HOME/.local/share/memora/memories.db"
export MEMORA_ALLOW_ANY_TAG=1        # Optional: allow custom tags
# export MEMORA_TAG_FILE=./tags.txt  # Optional: tag allowlist
```

## Verified Tools

### Serena Tools

| Tool | Purpose |
|------|---------|
| `find_symbol` | Locate symbols by name |
| `get_symbols_overview` | Get overview of all symbols in a file |
| `find_referencing_symbols` | Find all references to a symbol (callers) |
| `find_referenced_symbols` | Find all symbols referenced by a symbol (callees) |
| `replace_symbol_body` | Replace entire function/method body |
| `insert_before_symbol` | Insert code before a symbol |
| `insert_after_symbol` | Insert code after a symbol |
| `delete_lines` | Delete specific lines |
| `rename_symbol` | Rename and update all references |
| `write_memory` | Persist insights to `.serena/memories/` |
| `read_memory` | Retrieve previously stored memories |
| `onboarding` | Automated project familiarization |
| `think_about_task_adherence` | Check task alignment |

### Memora Tools

| Tool | Purpose |
|------|---------|
| `memory_create` | Store a new memory with content, tags, and section |
| `memory_search` | Hybrid search (FTS + semantic) across memories |
| `memory_list` | List memories with tag/section/date filters |
| `memory_update` | Update memory content or metadata |
| `memory_delete` | Remove a specific memory |
| `memory_get` | Retrieve a memory by ID |
| `memory_create_todo` | Create a TODO with status and priority |
| `memory_create_issue` | Create an issue with severity and component |
| `memory_link` | Create typed edge between two memories |
| `memory_find_duplicates` | Find similar memories for deduplication |
| `memory_merge` | Merge two memories |
| `memory_insights` | Activity summary, stale detection, consolidation suggestions |
| `memory_rebuild_embeddings` | Rebuild all embeddings after model change |

## Server Invocation

```bash
# Default (stdio mode for MCP)
memora-server

# With knowledge graph visualization
memora-server --graph-port 8765

# Headless (no graph server)
memora-server --no-graph

# HTTP transport (alternative to stdio)
memora-server --transport streamable-http --host 127.0.0.1 --port 8080
```

## Usage Examples

### Storing a Memory

```
memory_create(
    content="All database access goes through the repository layer. Direct ORM calls from handlers are forbidden.",
    tags=["architecture", "database"],
    section="decisions"
)
```

### Searching Context

```
memory_search(
    query="database access patterns",
    limit=5
)
```

## Air-Gap Compliance

| Check | Status | Action |
|-------|--------|--------|
| MEMORA_STORAGE_URI not set | Required | `unset MEMORA_STORAGE_URI` |
| CLOUDFLARE_API_TOKEN not set | Required | `unset CLOUDFLARE_API_TOKEN` |
| Local SQLite only | Required | Verify `MEMORA_DB_PATH` points to local path |
| No outbound network | Required | Firewall rules (container isolation is optional for air-gapped environments) |
| Embedding model pre-downloaded | Required | Download before going air-gapped |

## Performance Notes

- SQLite database scales well for moderate repositories
- Hybrid search (FTS5 + semantic) provides better recall than either method alone
- Session cleanup runs automatically based on timeout settings
- Embedding cache significantly speeds up repeated queries

## Related Documents

- [Memora Quickstart](./memora-quickstart.md)
- [Architecture Overview](../architecture/overview.md)
- [Security Assessment](../research/security-assessment.md)
- [Serena Quickstart](./serena-quickstart.md)
- [Srclight Quickstart](./srclight-quickstart.md)