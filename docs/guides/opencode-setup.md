# Crush Setup Guide

Crush is the **base AI coding agent** in this air-gapped architecture, providing terminal-based AI
assistance with plugin extensibility.

> **Note**: This project was originally called OpenCode. The `opencode-ai/opencode` repository was
> archived in September 2025. The active project is `charmbracelet/crush`. See the
> [Crush Quickstart](./opencode-quickstart.md) for installation.

## Overview

Crush acts as a 100% offline terminal coding agent, routing queries to local MCP servers and
coordinating specialized agents for codebase intelligence via the Oh-My-OpenCode-Slim plugin.

## Prerequisites

```bash
# MCP server dependencies (pre-download for air-gapped)
pip install "serena[mcp]"
pip install srclight
pip install "memora[local] @ git+https://github.com/agentic-box/memora.git"

# Ollama for local embeddings (required by Srclight)
# Download: https://ollama.com/download
ollama pull qwen3-embedding    # Best local quality (~6GB VRAM)
# ollama pull nomic-embed-text  # Lighter alternative (~4GB VRAM)

# Language servers (for C/C++)
apt-get install clangd  # or ccls

# Generate compile_commands.json
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
```

> **Air-Gap Note**: All packages and models must be downloaded and cached BEFORE disconnecting from the network.

## Oh-My-OpenCode-Slim Plugin

Oh-My-OpenCode-Slim (OMO Slim) is an OpenCode plugin that transforms single-agent OpenCode into a multi-agent orchestration system. It provides the hub-and-spoke architecture that coordinates all MCP servers (Serena, Srclight, Memora) through specialized agents.

### Installation

```bash
# Interactive installer (requires Bun)
bunx oh-my-opencode-slim@latest install

# Non-interactive (for scripting/air-gapped prep)
bunx oh-my-opencode-slim@latest install --no-tui --tmux=no --skills=yes
```

> **Air-Gap Warning**: `bunx` downloads from npm. For air-gapped environments, pre-install
> on a connected machine, cache the npm packages, then transfer via internal npm mirror (Verdaccio).

### Configuration

OMO Slim injects its configuration into Crush's config file (`.crush.json` or `~/.config/crush/crush.json`).
The schema is available for editor autocomplete:

```json
{
  "$schema": "https://unpkg.com/oh-my-opencode-slim@latest/oh-my-opencode-slim.schema.json",
  "agents": {
    "orchestrator": { "model": "openai/gpt-5.4" },
    "oracle": { "model": "openai/gpt-5.4" },
    "explorer": { "model": "openai/gpt-5.4-mini" },
    "librarian": { "model": "openai/gpt-5.4-mini" },
    "designer": { "model": "kimi-for-coding/k2p5" },
    "fixer": { "model": "openai/gpt-5.4-mini" }
  }
}
```

For local/air-gapped model routing, replace provider strings with your configured local provider:

```json
{
  "agents": {
    "orchestrator": { "model": "ollama/qwen3:30b" },
    "oracle": { "model": "ollama/qwen3:30b" },
    "explorer": { "model": "ollama/qwen3:7b" },
    "librarian": { "model": "ollama/qwen3:7b" },
    "designer": { "model": "ollama/qwen3:30b" },
    "fixer": { "model": "ollama/qwen3:7b" }
  }
}
```

### Specialized Agents

OMO Slim ships six agents. The names below are the actual agent identifiers in the plugin:

| Agent | Role | Default Model | Description |
|-------|------|---------------|-------------|
| Orchestrator | Coordinator | `gpt-5.4` | Main entry point. Delegates tasks to specialists. Determines optimal path to any goal. |
| Oracle | Advisor | `gpt-5.4` | Read-only consultant. Architecture review, debugging after 2+ failed attempts, complex tradeoffs. |
| Explorer | Recon | `gpt-5.4-mini` | Parallel codebase exploration. Contextual grep, pattern discovery. Runs in background. |
| Librarian | Knowledge | `gpt-5.4-mini` | External reference search. Official docs, OSS examples, GitHub code search. |
| Designer | Visual | `k2p5` | Frontend, UI/UX, design, styling, animation work. |
| Fixer | Implementation | `gpt-5.4-mini` | Fast implementation specialist. Bug fixes, quick changes, single-file modifications. |

> **Note**: This documentation project uses its own agent naming (Sisyphus, Prometheus, etc.) as
> aliases for the OMO Slim agents above. The underlying plugin agents are Orchestrator, Oracle,
> Explorer, Librarian, Designer, and Fixer.

### Pre-configured MCPs

OMO Slim includes three MCP servers out of the box:

| MCP | Purpose | Used By |
|-----|---------|---------|
| `websearch` | Real-time web search via Exa AI | Sisyphus, Librarian, Prometheus |
| `context7` | Official library documentation | Librarian |
| `grep_app` | GitHub code search via grep.app | Oracle |

### Task Categories

OMO Slim supports domain-specific task delegation:

| Category | Domain |
|----------|--------|
| `visual-engineering` | Frontend, UI/UX, design |
| `ultrabrain` | Hard logic-heavy problems |
| `deep` | Autonomous deep problem-solving |
| `artistry` | Creative, unconventional approaches |
| `quick` | Trivial single-file changes |
| `writing` | Documentation, prose |
| `business-logic` | General business logic |

### How It Changes the Interaction

Without OMO Slim, OpenCode runs as a single agent. With OMO Slim:
- Tasks are automatically delegated to the best-suited specialized agent
- Multiple agents can work in parallel (background tasks)
- Different LLM models are routed to different agent roles (cost optimization)
- Session lifecycle is managed through 25+ hooks
- MCP servers are coordinated through the orchestrator

## Directory Structure

```
project-root/
├── .crush.json          # Crush + OMO Slim configuration
├── .serena/             # Serena (symbolic layer)
│   ├── memories/        # Persistent insights
│   └── project.yml
├── .srclight/           # Srclight (semantic layer)
│   ├── index.db
│   └── config.yml
├── .memora/             # Memora (persistence layer)
│   └── memories.db
└── compile_commands.json
```

## Configuration

### MCP Server Configuration (.crush.json)

```json
{
  "$schema": "https://charm.land/crush.json",
  "mcp": {
    "serena": {
      "type": "stdio",
      "command": "uvx",
      "args": [
        "--from", "git+https://github.com/oraios/serena",
        "serena", "start-mcp-server",
        "--context", "ide-assistant", "--project", "."
      ]
    },
    "srclight": {
      "type": "stdio",
      "command": "srclight",
      "args": ["serve", "--workspace", "default"]
    },
    "memora": {
      "type": "stdio",
      "command": "memora-server",
      "env": {
        "MEMORA_DB_PATH": "~/.local/share/memora/memories.db",
        "MEMORA_EMBEDDING_MODEL": "sentence-transformers",
        "MEMORA_ALLOW_ANY_TAG": "1"
      }
    }
  },
  "options": {
    "disable_provider_auto_update": true
  }
}
```

> **Air-Gap Note**: Replace `git+https://` with locally installed packages.
> Pre-install: `pip install "serena[mcp]"` and use the local binary path.

## Query Routing

Crush routes queries to the appropriate MCP layer based on the task:

| Query Type | Handler | Example |
|------------|---------|---------|
| Symbol lookup | Serena | "Find function auth_user" |
| Natural language | Srclight | "Where is JSON parsing?" |
| Session context | Memora | "What did we decide about sessions?" |

## Workflow Example

```
1. Onboarding
   crush
   /init
   → Triggers Serena onboarding
   → Indexes with Srclight

2. Development
   User: "Find the login handler"
   → Serena: find_symbol("login_handler")

   User: "How does auth work?"
   → Srclight: search_hybrid("authentication flow")

   User: "What did we decide about sessions?"
   → Memora: memory_search("session management")

3. Memory
   User: "Remember: JWT tokens expire in 24h"
   → Memora: memory_create(content="JWT tokens expire in 24h", tags=["architecture"])
```

## Centralized Management

For team and enterprise deployments, OpenCode supports centralized patterns:

### Shared Ollama Server

Deploy a central Ollama instance serving embedding models to all team members:

```bash
# On the central server:
OLLAMA_HOST=0.0.0.0:11434 ollama serve

# Pre-load models:
ollama pull qwen3-embedding
ollama pull nomic-embed-text
```

Configure each developer's Srclight to point to the shared server:

```yaml
# .srclight/config.yml on each workstation
embeddings:
  provider: ollama
  host: http://ollama-server.internal:11434
  model: qwen3-embedding
```

### Configuration-as-Code

Distribute consistent tool configurations via version control:

```
team-dotfiles/
├── .opencode.json          # Shared OpenCode config
├── .serena/
│   └── project.yml         # Team-wide Serena settings
└── .srclight/
    └── config.yml          # Shared Srclight config
```

### Enterprise Deployment Pattern

```
┌─────────────────────────────────────────────┐
│            Internal Network                  │
│                                              │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐ │
│  │ Verdaccio│  │  devpi    │  │  Ollama   │ │
│  │ (npm)    │  │  (PyPI)   │  │  Server   │ │
│  └──────────┘  └───────────┘  └───────────┘ │
│       │             │              │         │
│  ┌────┴─────────────┴──────────────┴───────┐ │
│  │     Developer Workstations              │ │
│  │  OpenCode + Serena + Srclight + Memora  │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

1. **Verdaccio** (npm mirror): Hosts @oraios/serena and MCP dependencies
2. **devpi** (PyPI mirror): Hosts srclight, memora, sentence-transformers wheels
3. **Ollama Server**: Central embedding model serving
4. **Shared Git repos**: Team configurations and Srclight indexes

### Monitoring

- Track MCP server health via process monitoring
- Monitor Ollama GPU utilization for embedding workloads
- Log Serena onboarding times to detect performance issues
- Audit Memora database sizes per project

## Performance Notes

- First session requires onboarding (30s-2min)
- Subsequent sessions load from cache
- MCP server latency: ~50-200ms per call
- Agent coordination adds overhead; batch queries when possible

## Related Documents

- [Architecture Overview](../architecture/overview.md)
- [Serena Quickstart](./serena-quickstart.md)
- [Srclight Quickstart](./srclight-quickstart.md)
- [Memora Configuration](./memora-config.md)
- [Security Assessment](../research/security-assessment.md)