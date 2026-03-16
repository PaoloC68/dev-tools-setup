# Memora Configuration Guide

Memora is the **persistence layer** of the Oh-My-OpenCode-Slim stack, providing multi-session memory and context management for air-gapped development environments.

## Overview


Memora stores cross-session context, research findings, and architectural decisions using a local SQLite backend with semantic embeddings.

## Directory Structure

```
.serena/
├── memories/           # Serena's persistent insights (markdown/text)
└── project.yml        # Serena project configuration

.memora/
├── memory.db          # SQLite database for session data
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
# Database settings
database:
  path: .memora/memory.db
  auto_commit: true


# Embedding settings
embeddings:
  model: all-MiniLM-L6-v2
  device: cpu
  cache: .memora/embeddings/

# Context settings
context:
  max_sessions: 50
  session_timeout_days: 30
```

## Verified Tools

### Serena Tools

| Tool | Purpose |
|------|---------|
| `find_symbol` | Locate symbols by name |
| `get_symbols_overview` | Get overview of all symbols in a file |
| `find_referencing_symbols` | Find all references to a symbol |
| `replace_symbol_body` | Replace entire function/method body |
| `insert_before_symbol` | Insert code before a symbol |
| `insert_after_symbol` | Insert code after a symbol |
| `delete_lines` | Delete specific lines |
| `write_memory` | Persist insights to `.serena/memories/` |
| `onboarding` | Automated project familiarization |
| `think_about_task_adherence` | Check task alignment |

### Memora Tools

| Tool | Purpose |
|------|---------|
| `add_decision` | Record architectural decisions |
| `query_context` | Search past sessions |
| `get_session_history` | Retrieve session metadata |

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
query_context(
    query="authentication flow",
    limit=5
)
```

## Performance Notes

- SQLite database scales well for moderate repositories
- Embedding cache significantly speeds up repeated queries
- Session cleanup runs automatically based on timeout settings
