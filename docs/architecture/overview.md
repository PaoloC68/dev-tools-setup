# Oh-My-OpenCode-Slim Architecture

## Overview

Oh-My-OpenCode-Slim is a **100% offline, air-gapped RAG stack** for C/C++ codebase intelligence utilizing the Model Context Protocol (MCP) for local IPC.

## Architecture Stack (Hub-and-Spoke)

```
┌─────────────────────────────────────────────────────────────┐
│                     OpenCode (Hub)                          │
│              Orchestration & Agent Coordination            │
└─────────────────────────┬───────────────────────────────────┘
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Serena   │   │ Srclight │   │ Memora   │
    │(Symbolic)│   │(Semantic)│   │(History) │
    └──────────┘   └──────────┘   └──────────┘
```

### 1. OpenCode — Orchestration Layer

**Purpose**: Central IDE/Orchestration hub (air-gapped replacement for Claude Code).

- Routes queries to appropriate MCP servers
- Coordinates six specialized agents:
  - `architect` — System design
  - `implementer` — Code generation
  - `reviewer` — Code review
  - `debugger` — Troubleshooting
  - `researcher` — Information gathering
  - `security` — Vulnerability analysis

### 2. Serena — Symbolic/Structural Layer

**Purpose**: LSP-based structural navigation and scope-aware code modification.

- **Backend**: clangd (default) or ccls
- **Languages**: 30+ languages via multilspy
- **Storage**: `.serena/memories/` (markdown/text files)

#### Verified Tools

| Category | Tools |
|----------|-------|
| Navigation | `find_symbol`, `get_symbols_overview`, `find_referencing_symbols` |
| Editing | `replace_symbol_body`, `insert_before_symbol`, `insert_after_symbol`, `delete_lines` |
| Meta | `write_memory`, `onboarding`, `think_about_task_adherence` |

### 3. Srclight — Semantic Layer

**Purpose**: Air-gap certified code indexer with vector database for natural language search.

- **Model**: all-MiniLM-L6-v2 (local embeddings)
- **Database**: SQLite (`.srclight/index.db`)
- **Offline**: Mandatory `offline_mode: true`

### 4. Memora — Persistence Layer

**Purpose**: Multi-session data persistence and cross-session context.

- **Backend**: SQLite (`.memora/memory.db`)
- **Embeddings**: all-MiniLM-L6-v2
- **Data**: Session metadata, research, TODOs (often Git-linked)

## Directory Structure

```
project-root/
├── .opencode/           # OpenCode config
├── .serena/             # Serena (symbolic)
│   ├── memories/       # Persistent insights
│   └── project.yml
├── .srclight/           # Srclight (semantic)
│   ├── index.db
│   └── config.yml
├── .memora/             # Memora (persistence)
│   └── memory.db
├── compile_commands.json
└── opencode-workspace/
```

## Integration Modes

All components support three integration patterns:

| Mode | Description |
|------|-------------|
| MCP Server | Standalone MCP server for 100% offline |
| Agno Agent | Integrated as agent tool |
| Framework | Direct import for custom frameworks |

## Data Flow

```
User Query
    │
    ▼
┌─────────────────┐
│    OpenCode     │
│  (Orchestrator) │
└────────┬────────┘
         │
    ┌────┼────┐
    ▼    ▼    ▼
Serena Srclight Memora
(Sym)  (Sem)  (Hist)
    │    │    │
    └────┼────┘
         ▼
   Response + Memory
```

## Key Distinctions

| Layer | Storage | Query Type | Example |
|-------|---------|------------|---------|
| Serena | `.serena/memories/` (markdown) | Symbolic | "Find function login" |
| Srclight | `.srclight/index.db` (vector) | Semantic | "Where is auth handled?" |
| Memora | `.memora/memory.db` (SQLite) | Historical | "What did we decide?" |

## Performance

- **Token Efficiency**: ~70% savings over text-based RAG
- **MCP Latency**: ~50-200ms per call
- **Onboarding**: 30s-2min (first session)
- **Subsequent**: Load from cache

## Requirements

- `compile_commands.json` at repository root
- Pre-downloaded embedding models
- Language servers (clangd/ccls) installed
- 100% offline operation
