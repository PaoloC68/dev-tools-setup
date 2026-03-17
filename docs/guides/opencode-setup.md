# OpenCode Setup Guide

OpenCode is the **base AI coding agent** in this air-gapped architecture, providing terminal-based AI assistance with plugin extensibility.

## Overview

OpenCode acts as a 100% offline IDE replacement, routing queries to localized MCP servers and coordinating six specialized agents for codebase intelligence.

## Prerequisites

```bash
# Core dependencies (pre-download for air-gapped)
pip install opencode serena srclight memora

# Ollama for local embeddings (required by Srclight)
# Download: https://ollama.com/download
ollama pull qwen3-embedding    # Best local quality (~6GB VRAM)
# ollama pull nomic-embed-text  # Lighter alternative (8GB VRAM)

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

Config file: `~/.config/opencode/oh-my-opencode-slim.json` (or `.jsonc` for comments)

```json
{
  "$schema": "https://raw.githubusercontent.com/alvinunreal/oh-my-opencode-slim/master/assets/oh-my-opencode-slim.schema.json",
  "agents": {
    "sisyphus": { "model": "anthropic/claude-opus-4" },
    "prometheus": { "model": "anthropic/claude-opus-4" },
    "oracle": { "model": "anthropic/claude-opus-4" },
    "explorer": { "model": "anthropic/claude-haiku-3.5" },
    "librarian": { "model": "anthropic/claude-haiku-3.5" },
    "designer": { "model": "anthropic/claude-sonnet-4" },
    "fixer": { "model": "anthropic/claude-sonnet-4" }
  }
}
```

### Specialized Agents

| Agent | Role | Typical Model | Description |
|-------|------|---------------|-------------|
| Sisyphus | Orchestrator | Opus-class | Main entry point. Delegates tasks to specialists. Never works alone when delegation is possible. |
| Prometheus | Planner | Opus-class | Creates parallel task graphs, structured TODO lists. Invoked before complex implementations. |
| Oracle | Advisor | Opus-class | Read-only consultant. Architecture review, debugging after 2+ failed attempts, complex tradeoffs. |
| Explorer | Recon | Haiku-class | Parallel codebase exploration. Contextual grep, pattern discovery. Runs in background. Cheap and fast. |
| Librarian | Knowledge | Haiku-class | External reference search. Official docs, OSS examples, GitHub code search. |
| Designer | Visual | Sonnet-class | Frontend, UI/UX, design, styling, animation work. |
| Fixer | Implementation | Sonnet-class | Bug fixes, quick changes, single-file modifications. |

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
├── .opencode/           # OpenCode configuration
│   └── config.yml
├── .serena/             # Serena (symbolic layer)
│   ├── memories/       # Persistent insights
│   └── project.yml
├── .srclight/           # Srclight (semantic layer)
│   ├── index.db
│   └── config.yml
├── .memora/             # Memora (persistence layer)
│   └── memory.db
├── compile_commands.json
└── opencode-workspace/
```

## Configuration


### OpenCode Config (.opencode/config.yml)

```yaml
# Orchestration settings
orchestration:
  mode: local           # Always local for air-gap
  mcp_servers:
    - serena
    - srclight
    - memora

# Agent configuration
agents:
  - name: architect
    role: system_design
  - name: implementer
    role: code_generation
  - name: reviewer
    role: code_review
  - name: debugger
    role: troubleshooting
  - name: researcher
    role: information_gathering
  - name: security
    role: vulnerability_analysis

# Network (always offline)
network:
  offline_mode: true
  proxy: null
```

### MCP Server Configuration (~/.opencode.json)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["oh-my-opencode-slim@latest"],
  "mcp": {
    "serena": {
      "type": "local",
      "command": ["uvx", "--from", "git+https://github.com/oraios/serena",
                  "serena", "start-mcp-server",
                  "--context", "ide-assistant", "--project", "."],
      "enabled": true
    },
    "srclight": {
      "type": "local",
      "command": ["srclight", "serve", "--workspace", "default"],
      "enabled": true
    }
  }
}
```

> **Air-Gap Note**: For air-gapped environments, replace `git+https://` with locally installed packages.
> Pre-install: `pip install serena` or `npm install -g @oraios/serena`

## Integration Modes


OpenCode supports three integration patterns:

### 1. MCP Server Mode

Standalone MCP server for 100% offline deployments:

```bash
opencode serve --port 3000
```

### 2. Agno Agent Mode

Integrated as an agent tool:

```python
from agno import Agent
agent = Agent(tools=[opencode_context])
```

### 3. Framework Integration

Direct import for custom frameworks:

```python
from opencode import Orchestrator
orch = Orchestrator()
```

## Usage

### Starting a Session

```bash
# Initialize with project
opencode init /path/to/project

# Start interactive session
opencode chat
```

### Query Routing

OpenCode automatically routes queries to the appropriate layer:

| Query Type | Handler | Example |
|------------|---------|----------|
| Symbol lookup | Serena | "Find function auth_user" |
| Natural language | Srclight | "Where is JSON parsing?" |
| Session context | Memora | "What did we discuss about auth?" |

### Agent Coordination

```python
# Delegate to specialized agent
delegate to architect: "Design a caching layer"
delegate to reviewer: "Review auth_module.cpp"
```

## Workflow Example


```
1. Onboarding
   opencode init /project
   → Triggers Serena onboarding
   → Indexes with Srclight

2. Development
   User: "Find the login handler"
   → Serena: find_symbol("login_handler")
   
   User: "How does auth work?"
   → Srclight: semantic_search("authentication flow")
   
   User: "What did we decide about sessions?"
   → Memora: query_context("session management")

3. Memory
   User: "Remember this insight"
   → Serena: write_memory("JWT tokens expire in 24h")
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