# OpenCode Setup Guide

OpenCode is the **orchestration layer** of the Oh-My-OpenCode-Slim stack, serving as the central hub that coordinates Serena, Srclight, and Memora for air-gapped development.

## Overview

OpenCode acts as a 100% offline IDE replacement, routing queries to localized MCP servers and coordinating six specialized agents for codebase intelligence.

## Prerequisites

```bash
# Core dependencies
pip install opencode serena srclight memora

# Language servers (for C/C++)
apt-get install clangd  # or ccls

# Generate compile_commands.json
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..
```

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

## Performance Notes

- First session requires onboarding (30s-2min)
- Subsequent sessions load from cache
- MCP server latency: ~50-200ms per call
- Agent coordination adds overhead; batch queries when possible
