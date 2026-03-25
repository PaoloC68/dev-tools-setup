# OpenCode Setup Guide

OpenCode is the **base AI coding agent** in this air-gapped architecture, providing terminal-based AI assistance with plugin extensibility.

## Overview

OpenCode acts as a 100% offline IDE replacement, routing queries to localized MCP servers and coordinating six specialized agents for codebase intelligence.

## Prerequisites

```bash
# Install OpenCode
curl -fsSL https://opencode.ai/install | bash
# or: npm i -g opencode-ai@latest

# Install MCP server dependencies (pre-download for air-gapped)
pip install "serena[mcp]"
pip install srclight
pip install "memora-mcp[local] @ git+https://github.com/agentic-box/memora.git"

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

OMO Slim is loaded as an OpenCode plugin. Add it to your `opencode.json`:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "plugin": ["oh-my-opencode-slim@latest"]
}
```

OMO Slim's own agent configuration lives in `~/.config/opencode/oh-my-opencode-slim.json`
(global) or `.opencode/oh-my-opencode-slim.json` (project-local). Both support `.jsonc`
(JSON with comments). The schema provides editor autocomplete:

```jsonc
{
  "$schema": "https://unpkg.com/oh-my-opencode-slim@latest/oh-my-opencode-slim.schema.json",

  "preset": "airgap",

  "presets": {
    "airgap": {
      // Tier 1: Best reasoning (orchestration, review, planning)
      "orchestrator": { "model": "Qwen3.5-397B-A17B",         "skills": ["*"], "mcps": [] },
      "oracle":       { "model": "Qwen3.5-397B-A17B",         "skills": [], "mcps": [] },

      // Tier 2: Fast (recon and retrieval)
      "explorer":     { "model": "qwen3-coder-next",           "skills": [], "mcps": [] },
      "librarian":    { "model": "Qwen3-30B-A3B-Instruct-2507","skills": [], "mcps": [] },

      // Tier 3: Balanced (implementation and design)
      "designer":     { "model": "Kimi-K2.5-NoReasoning",     "skills": [], "mcps": [] },
      "fixer":        { "model": "gpt-oss-120b-thinking-low",  "skills": [], "mcps": [] }
    }
  },

  // Disable all outbound MCPs (air-gapped environment)
  "disabled_mcps": ["websearch", "context7", "grep_app"]
}
```

Model IDs must match exactly what your internal inference server exposes. See
[Model-Agent Mapping](../research/model-agent-mapping.md) for rationale and alternative
configurations (throughput-optimized, maximum context).

### Specialized Agents

OMO Slim ships six agents. These are the actual agent identifiers used in configuration:

> **Note**: The OMO Slim installer generates OpenAI defaults (`openai/gpt-5.4`, etc.) which
> are unavailable in this air-gapped deployment. Always override with models from the internal
> inference server. See [Model-Agent Mapping](../research/model-agent-mapping.md) for the
> full recommended configuration.

| Agent | Role | Recommended Model |
|-------|------|-------------------|
| `orchestrator` | Master delegator and strategic coordinator | `Qwen3.5-397B-A17B` |
| `oracle` | Strategic advisor and debugger of last resort. Read-only. | `Qwen3.5-397B-A17B` |
| `librarian` | External knowledge retrieval. Docs, OSS examples, search. | `Qwen3-30B-A3B-Instruct-2507` |
| `explorer` | Codebase reconnaissance. Fast, parallel, read-only. | `qwen3-coder-next` |
| `designer` | UI/UX implementation and visual excellence. | `Kimi-K2.5-NoReasoning` |
| `fixer` | Fast implementation specialist. Bug fixes, single-file changes. | `gpt-oss-120b-thinking-low` |

### Pre-configured MCPs

OMO Slim includes three MCP servers out of the box. Each agent has a default allowlist:

| MCP | URL | Default Agents |
|-----|-----|----------------|
| `websearch` | `https://mcp.exa.ai/mcp` | `orchestrator`, `librarian` |
| `context7` | `https://mcp.context7.com/mcp` | `librarian` |
| `grep_app` | `https://mcp.grep.app` | `librarian` |

> **Air-Gap Warning**: All three MCPs make outbound network calls. Disable them in
> `oh-my-opencode-slim.json` (the OMO Slim config, not `opencode.json`):
> ```json
> { "disabled_mcps": ["websearch", "context7", "grep_app"] }
> ```
> This is already included in the full config example above.

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
- MCP servers are coordinated through the orchestrator

## Directory Structure

```
project-root/
├── opencode.json                              # OpenCode config
├── .opencode/
│   └── oh-my-opencode-slim.json              # OMO Slim project-local config (optional)
├── AGENTS.md                                  # Project context (generated by /init)
├── .serena/
│   ├── memories/                              # Serena persistent insights
│   └── project.yml
├── .srclight/
│   └── index.db                               # Srclight index (no config file)
└── compile_commands.json

~/.config/opencode/
├── opencode.json                              # OpenCode global config
└── oh-my-opencode-slim.json                  # OMO Slim global config
```

## Configuration

### OpenCode Config (opencode.json)

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "internal": {
      "name": "Internal",
      "api": "openai",
      "url": "http://inference.internal/v1",
      "options": {
        "apiKey": "{env:INFERENCE_API_KEY}"
      }
    }
  },
  "model": "internal/your-llm-model",
  "plugin": ["oh-my-opencode-slim@latest"],
  "autoupdate": false,
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
      "command": ["srclight", "serve", "--workspace", "myworkspace", "--transport", "stdio"],
      "enabled": true
    },
    "memora": {
      "type": "local",
      "command": ["memora-server"],
      "enabled": true,
      "environment": {
        "MEMORA_DB_PATH": "~/.local/share/memora/memories.db",
        "MEMORA_EMBEDDING_MODEL": "sentence-transformers",
        "MEMORA_ALLOW_ANY_TAG": "1"
      }
    }
  }
}
```

> **Air-Gap Note**: Replace `git+https://` with locally installed packages.
> Pre-install Serena via `pip install "serena[mcp]"` and use the local binary path.
> Set `"autoupdate": false` to prevent OpenCode from checking for updates at startup.

### Config File Locations (precedence order)

1. Remote (`.well-known/opencode`) — organizational defaults
2. Global (`~/.config/opencode/opencode.json`) — user preferences
3. Project (`opencode.json` in project root) — project-specific settings

Configs are **merged**, not replaced. Project settings override global ones for conflicting keys.

## Usage

### Starting a Session

```bash
cd /path/to/project
opencode

# Initialize project context (first time)
/init

# Run a single prompt non-interactively
opencode run "Explain the authentication flow"
```

### Query Routing

OpenCode routes queries to the appropriate MCP layer:

| Query Type | Handler | Example |
|------------|---------|---------|
| Symbol lookup | Serena | "Find function auth_user" |
| Natural language | Srclight | "Where is JSON parsing?" |
| Session context | Memora | "What did we decide about sessions?" |

## Workflow Example

```
1. Onboarding
   opencode
   /init
   → OpenCode analyzes project, creates AGENTS.md
   → Serena indexes symbols via LSP
   → Srclight indexes codebase for hybrid search

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

### Configuration-as-Code

Distribute consistent tool configurations via version control:

```
team-dotfiles/
├── opencode.json                          # Shared OpenCode config
├── oh-my-opencode-slim.json              # Shared OMO Slim agent config
└── .serena/
    └── project.yml                        # Team-wide Serena settings
```

### Enterprise Deployment Pattern

```
┌─────────────────────────────────────────────┐
│            Internal Network                  │
│                                              │
│  ┌──────────┐  ┌───────────┐  ┌───────────┐ │
│  │ Verdaccio│  │  devpi    │  │ Inference │ │
│  │ (npm)    │  │  (PyPI)   │  │  Server   │ │
│  └──────────┘  └───────────┘  └───────────┘ │
│       │             │              │         │
│  ┌────┴─────────────┴──────────────┴───────┐ │
│  │     Developer Workstations              │ │
│  │  OpenCode + Serena + Srclight + Memora  │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

1. **Verdaccio** (npm mirror): Hosts oh-my-opencode-slim and MCP dependencies
2. **devpi** (PyPI mirror): Hosts srclight, memora, serena wheels
3. **Inference Server**: Central LLM and embedding model serving (OpenAI-compatible API)
4. **Shared Git repos**: Team configurations and Srclight indexes

### Monitoring

- Track MCP server health via process monitoring
- Monitor inference server latency for embedding workloads
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