# Air-Gapped AI-Assisted Coding Architecture

## Overview

This architecture is a **100% offline, air-gapped stack** for AI-assisted coding and C/C++ codebase intelligence. It combines a terminal-based AI coding agent, a multi-agent orchestration plugin, and three specialized MCP servers into a single cohesive system. All components operate locally with zero runtime network access.

The stack uses the Model Context Protocol (MCP) for local IPC between the orchestration layer and backend servers.

## Component Hierarchy

Five components, layered from user-facing agent down to backend intelligence servers:

| # | Component | Role | Repository |
|---|-----------|------|------------|
| 1 | **OpenCode** | Base terminal AI coding agent | anomalyco/opencode |
| 2 | **Oh-My-OpenCode-Slim** | Multi-agent orchestration plugin | alvinunreal/oh-my-opencode-slim |
| 3 | **Serena** | Symbolic/structural MCP server | oraios/serena |
| 4 | **Srclight** | Semantic search MCP server | srclight/srclight |
| 5 | **Memora** | Persistence MCP server | agentic-mcp-tools/memora |

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode (Base Agent)                     │
│              Terminal AI Coding Assistant                    │
├─────────────────────────────────────────────────────────────┤
│              Oh-My-OpenCode-Slim (Plugin)                   │
│         Multi-Agent Orchestration & Task Delegation          │
│                                                             │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│  │Sisyphus  │ │Prometheus│ │ Oracle   │ │ Explorer │      │
│  │(Orchestr)│ │(Planner) │ │(Advisor) │ │(Recon)   │      │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐                   │
│  │Librarian │ │ Designer │ │  Fixer   │                   │
│  │(Knowledge│ │(Visual)  │ │(Impl.)   │                   │
│  └──────────┘ └──────────┘ └──────────┘                   │
└─────────────────────────┬───────────────────────────────────┘
                          │ MCP (stdio)
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │ Serena   │   │ Srclight │   │ Memora   │
    │(Symbolic)│   │(Semantic)│   │(Memory)  │
    └──────────┘   └──────────┘   └──────────┘
```

## 1. OpenCode (Base Agent)

**Purpose**: Open-source terminal-based AI coding assistant. The foundation layer that everything else builds on.

- Supports multiple LLM providers (local and remote)
- Plugin system for extending functionality
- Custom commands and slash-command support
- Native MCP server integration
- Config at `~/.config/opencode/opencode.json`

OpenCode on its own is a capable single-agent coding assistant. The orchestration plugin transforms it into something more powerful.

## 2. Oh-My-OpenCode-Slim (Orchestration Plugin)

**Purpose**: Transforms OpenCode from a single-agent tool into a multi-agent orchestration system. This is a plugin, not a standalone application.

- **Install**: `bunx oh-my-opencode-slim@latest install`
- **Config**: `~/.config/opencode/oh-my-opencode-slim.json` (JSONC format, comments supported)
- **Air-Gap Note**: Pre-install before disconnecting from the network. For fully offline environments, populate the npm/bun cache while connected, then install from cache.

### Hub-and-Spoke Model

OMO Slim implements a hub-and-spoke architecture where the Sisyphus agent acts as the central orchestrator, delegating work to six specialized agents. Each agent can use different LLM models depending on the task complexity.

### Agent Roster

| Agent | Role | Description |
|-------|------|-------------|
| **Sisyphus** | Orchestrator | Supreme executor and delegator. Routes tasks to specialized agents. Main entry point. |
| **Prometheus** | Planner | Strategic planning. Analyzes requests, creates parallel task graphs, structured TODO lists. |
| **Oracle** | Strategic Advisor | Read-only high-IQ consultant. Architecture review, debugging, complex logic. |
| **Explorer** | Codebase Recon | Parallel codebase exploration. Contextual grep, pattern discovery. Runs in background. |
| **Librarian** | Knowledge Retrieval | External references: official docs, OSS examples, GitHub search. Uses websearch, context7, grep_app MCPs. |
| **Designer** | Visual Excellence | Frontend, UI/UX, design, styling, animation. |
| **Fixer** | Implementation | Bug fixes, quick changes, single-file modifications. |

### Key Features

- **Multi-model routing**: Assign different LLM models to different agent roles. Heavy reasoning tasks get a stronger model; quick fixes get a faster one.
- **Background task management**: Fire parallel agents for concurrent exploration and implementation.
- **Session hooks**: 25+ lifecycle hooks (session.created, session.compacted, message.sent, etc.) for custom automation.
- **Pre-configured MCPs**: websearch (Exa), context7 (documentation lookup), grep_app (GitHub code search).
- **Built-in skills**: playwright (browser automation), git-master (atomic commits, rebase, history search).
- **Task categories**: visual-engineering, ultrabrain, deep, artistry, quick, writing, business-logic. Each category can route to a different model configuration.

## 3. Serena (Symbolic/Structural MCP Server)

**Purpose**: LSP-based structural navigation and scope-aware code modification.

- **Backend**: clangd (default, recommended) or ccls (better for repos >50k files)
- **Languages**: 30+ languages via multilspy
- **Storage**: `.serena/memories/` (markdown/text files)

### Verified Tools

| Category | Tools |
|----------|-------|
| Navigation | `find_symbol`, `get_symbols_overview`, `find_referencing_symbols` |
| Editing | `replace_symbol_body`, `insert_before_symbol`, `insert_after_symbol`, `delete_lines` |
| Meta | `write_memory`, `onboarding`, `think_about_task_adherence` |

## 4. Srclight (Semantic Search MCP Server)

**Purpose**: Air-gap certified code indexer with hybrid search combining keyword and semantic retrieval.

- **Parsing**: tree-sitter (precise symbol extraction: functions, classes, methods, interfaces, structs)
- **Languages**: C, C++, Python, TypeScript, JavaScript, Rust, Go
- **Keyword Search**: SQLite FTS5 with trigram + porter stemmer
- **Semantic Search**: Local embeddings via Ollama (qwen3-embedding default, nomic-embed-text alternative)
- **Hybrid Search**: Reciprocal Rank Fusion combining FTS5 + semantic results
- **Database**: SQLite (`.srclight/index.db`)
- **Tools**: 25 MCP tools covering symbol search, relationship graphs, git change intelligence, semantic search, build system awareness, and document extraction
- **Offline**: Requires pre-downloaded Ollama models; no runtime network access

## 5. Memora (Persistence MCP Server)

**Purpose**: Multi-session data persistence and cross-session context.

- **Backend**: SQLite (local-only mode for air-gapped; cloud sync disabled)
- **Search**: Hybrid keyword + semantic search with Reciprocal Rank Fusion
- **Data**: Session metadata, research, TODOs, cross-session context
- **Air-Gap Note**: Cloud sync features (S3, R2, D1) exist but MUST be disabled for air-gapped deployments

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

Global configs:

```
~/.config/opencode/
├── opencode.json                    # OpenCode base config
└── oh-my-opencode-slim.json         # OMO Slim plugin config (JSONC)
```

## SOTA Comparison

This architecture represents the current state-of-the-art for air-gapped AI coding assistance. Comparison with alternatives:

| Solution | Offline | Code Context | LSP Integration | MCP Support | Open Source |
|----------|---------|-------------|-----------------|-------------|-------------|
| **This Architecture** | 100% air-gapped | Hybrid (symbolic + semantic + memory) | Serena (30+ languages) | Native | Yes |
| Tabnine Enterprise | Air-gapped option | Proprietary RAG | Limited | No | No |
| Continue.dev | Local LLM support | File-based context | Basic | Partial | Yes |
| Tabby (TabbyML) | Self-hosted | Repository-level | No | No | Yes |
| Sourcegraph Cody | Self-hosted option | Code graph | Sourcegraph search | No | Partial |
| Aider | Local LLM support | File-based diffs | No | No | Yes |

### Architectural Advantages

- **Three-layer context**: Symbolic (Serena) + Semantic (Srclight) + Historical (Memora) provides richer context than single-layer approaches
- **Multi-agent orchestration**: Seven specialized agents via OMO Slim, each optimized for a specific task type
- **Hub-and-spoke decoupling**: Each component independently upgradeable without affecting others
- **MCP standardization**: Linux Foundation-backed protocol ensures interoperability
- **Token efficiency**: ~70% savings over text-based RAG via symbol-level retrieval

## Scalability

### Horizontal Scaling

| Component | Scaling Strategy | Large Codebase Support |
|-----------|-----------------|----------------------|
| Serena | ccls for repos >50k files; per-language LSP instances | Restart LSP after adding files; pre-index nightly |
| Srclight | Multi-repo ATTACH across SQLite databases; UNION queries | Batch rotation for >10 ATTACH limit; GPU-accelerated embeddings via CuPy |
| Memora | Single SQLite per project; session cleanup via timeout | Configurable session limits (default 50) |

### Performance Characteristics

- **Serena**: Onboarding 30s-2min; subsequent loads from cache
- **Srclight**: ~1000 files/minute indexing; <100ms search (CPU); sub-3ms with GPU cache
- **Memora**: SQLite scales well for moderate repositories
- **MCP Latency**: ~50-200ms per call

## Centralized Management

For team and enterprise deployments, the architecture supports centralized patterns:

### Shared Model Serving

```
┌──────────────────────────────────┐
│     Ollama Server (Central)      │
│   - qwen3-embedding              │
│   - Code LLM (e.g. Qwen2.5)     │
└──────────────┬───────────────────┘
               │ HTTP (internal network)
    ┌──────────┼──────────┐
    ▼          ▼          ▼
 Dev A      Dev B      Dev C
 (OpenCode) (OpenCode) (OpenCode)
```

### Configuration-as-Code

- Version control `.serena/project.yml` for team-wide LSP settings
- Shared `.srclight/` configuration via repository
- Centralized Ollama model registry for consistent embeddings
- OpenCode config (`~/.config/opencode/opencode.json`) distributable via dotfiles repo
- OMO Slim config (`~/.config/opencode/oh-my-opencode-slim.json`) shareable for consistent agent behavior

### Enterprise Deployment Pattern

1. **Internal artifact mirrors**: Verdaccio (npm), devpi (PyPI) for offline package management
2. **Pre-built container images**: Docker/Podman with all dependencies pre-installed
3. **Shared vector indexes**: Srclight databases shareable across team members via internal storage
4. **Centralized LSP**: Language servers deployable as shared services

## Key Distinctions

| Layer | Storage | Query Type | Example |
|-------|---------|------------|---------|
| Serena | `.serena/memories/` (markdown) | Symbolic | "Find function login" |
| Srclight | `.srclight/index.db` (SQLite + FTS5 + embeddings) | Hybrid (keyword + semantic) | "Where is auth handled?" |
| Memora | local SQLite (cloud sync disabled) | Historical + semantic | "What did we decide?" |

## Data Flow

```
User Query
    │
    ▼
┌──────────────────────┐
│      OpenCode        │
│    (Base Agent)      │
├──────────────────────┤
│   OMO Slim Plugin    │
│  Sisyphus routes to  │
│  specialized agents  │
└──────────┬───────────┘
           │
      ┌────┼────┐
      ▼    ▼    ▼
  Serena Srclight Memora
  (Sym)  (Sem)   (Hist)
      │    │    │
      └────┼────┘
           ▼
     Response + Memory
```

## Performance

- **Token Efficiency**: ~70% savings over text-based RAG
- **MCP Latency**: ~50-200ms per call
- **Onboarding**: 30s-2min (first session)
- **Subsequent**: Load from cache

## Requirements

- `compile_commands.json` at repository root (C/C++ projects)
- Ollama with pre-downloaded embedding models (qwen3-embedding or nomic-embed-text)
- Language servers (clangd/ccls) installed locally
- Bun runtime for OMO Slim installation (`bunx oh-my-opencode-slim@latest install`)
- Python 3.10+ for Srclight
- 100% offline operation after initial setup

## Related Documents

- [OpenCode Quickstart](../guides/opencode-quickstart.md)
- [OpenCode Setup](../guides/opencode-setup.md)
- [Serena Quickstart](../guides/serena-quickstart.md)
- [Srclight Quickstart](../guides/srclight-quickstart.md)
- [Srclight Setup](../guides/srclight-setup.md)
- [Memora Configuration](../guides/memora-config.md)
- [Security Assessment](../research/security-assessment.md)
- [Serena LSP Best Practices](../research/serena-lsp-guide.md)