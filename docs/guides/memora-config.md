# Memora Configuration Guide

Memora is the **persistence layer** of the air-gapped AI coding architecture, providing multi-session memory and context management for air-gapped development environments.

## Overview

Memora stores cross-session context, research findings, and architectural decisions using a local SQLite backend with hybrid keyword + semantic search.

> **Air-Gap Warning**: Memora includes optional cloud sync features (S3, R2, Cloudflare D1).
> These MUST remain disabled for air-gapped deployments. See [Security Configuration](#security-configuration) below.

## Directory Structure

```
.serena/
├── memories/           # Serena's persistent insights (markdown/text)
└── project.yml        # Serena project configuration

.memora/
├── memories.db        # SQLite database for session data
└── embeddings/        # Local embedding cache
```

> **Note**: Memory files are stored in `.serena/memories/` (not `.memora/`).

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

# LSP settings
lsp:
  server: clangd          # or ccls
  compile_commands: ./compile_commands.json
```

### Memora Configuration

```yaml
# Database settings (local-only for air-gapped)
database:
  path: .memora/memories.db
  auto_commit: true

# Embedding settings
embeddings:
  provider: local            # Use local embeddings only
  model: sentence-transformers  # or ollama
  device: cpu
  cache: .memora/embeddings/

# Context settings
context:
  max_sessions: 50
  session_timeout_days: 30

# Search
search:
  hybrid: true              # FTS5 keyword + semantic
  fusion: reciprocal_rank   # Reciprocal Rank Fusion
```

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
| `add_memories` | Store new memories with metadata and tags |
| `search_memory` | Hybrid search (keyword + semantic) across memories |
| `list_memories` | List stored memories with filters |
| `delete_memory` | Remove specific memories |

## Integration Modes


Memora supports three integration patterns:

### 1. MCP Server Mode

Standalone MCP server for 100% offline deployments:

```bash
memora serve --port 3001
```

### 2. Agno Agent Mode

Integrated as an agent tool:

```python
from agno import Agent
agent = Agent(tools=[memora_context])
```

### 3. Framework Integration

Direct import for custom frameworks:

```python
from memora import ContextManager
ctx = ContextManager()
```

## Usage Examples

### Writing Memory (Serena)

```python
write_memory(
    content="# Key Insight\n\nThis module handles authentication via JWT tokens.",
    category="architecture"
)
```

### Querying Context (Memora)

```python
search_memory(
    query="authentication flow",
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

- [Architecture Overview](../architecture/overview.md)
- [Security Assessment](../research/security-assessment.md)
- [Serena Quickstart](./serena-quickstart.md)
- [Srclight Quickstart](./srclight-quickstart.md)