# Crush Quickstart Guide

## What is Crush?

**Crush** (github.com/charmbracelet/crush) is the terminal AI coding agent at the base of this
architecture. Originally developed as OpenCode, it was adopted by the Charm team in September 2025
and continues under the name Crush.

> **Note**: The original `opencode-ai/opencode` repository is archived. All active development
> is at `charmbracelet/crush`. The install commands, config format, and binary name have changed.

### Key Features

- **Multi-Model**: Any LLM via OpenAI- or Anthropic-compatible APIs, plus built-in providers
- **MCP Support**: `stdio`, `http`, and `sse` transport types
- **LSP Integration**: Uses language servers for additional code context
- **Session-Based**: Multiple work sessions per project
- **Agent Skills**: Extensible via the [Agent Skills](https://agentskills.io) open standard
- **100% Offline**: Local models via Ollama or LM Studio with `openai-compat` provider type

## Installation

### Package Managers (Recommended)

```bash
# macOS / Linux
brew install charmbracelet/tap/crush

# npm
npm install -g @charmland/crush

# Arch Linux
yay -S crush-bin

# Debian / Ubuntu
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://repo.charm.sh/apt/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/charm.gpg
echo "deb [signed-by=/etc/apt/keyrings/charm.gpg] https://repo.charm.sh/apt/ * *" | sudo tee /etc/apt/sources.list.d/charm.list
sudo apt update && sudo apt install crush

# Windows
winget install charmbracelet.crush
```

### Go

```bash
go install github.com/charmbracelet/crush@latest
```

> **Air-Gap Note**: Download the binary or package before disconnecting. Crush auto-updates
> its provider list from Catwalk on startup — disable this for air-gapped environments
> (see Configuration below).

## Configuration

### Config File Locations (priority order)

1. `.crush.json` (project root)
2. `crush.json` (project root)
3. `~/.config/crush/crush.json` (global)

### Minimal Config with MCP Servers

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
        "MEMORA_EMBEDDING_MODEL": "sentence-transformers"
      }
    }
  }
}
```

> **Air-Gap Note**: Replace `git+https://` with locally installed packages. Pre-install Serena
> via `pip install serena` and use the local binary path. Disable provider auto-updates (see below).

### Air-Gap Settings

```json
{
  "$schema": "https://charm.land/crush.json",
  "options": {
    "disable_provider_auto_update": true
  }
}
```

Or via environment variable:

```bash
export CRUSH_DISABLE_PROVIDER_AUTO_UPDATE=1
```

### Local Model (Ollama)

```json
{
  "$schema": "https://charm.land/crush.json",
  "providers": {
    "ollama": {
      "name": "Ollama",
      "base_url": "http://localhost:11434/v1/",
      "type": "openai-compat",
      "models": [
        {
          "name": "Qwen 3 30B",
          "id": "qwen3:30b",
          "context_window": 256000,
          "default_max_tokens": 20000
        }
      ]
    }
  }
}
```

### Environment Variables

| Variable | Provider |
|----------|----------|
| `ANTHROPIC_API_KEY` | Anthropic |
| `OPENAI_API_KEY` | OpenAI |
| `GEMINI_API_KEY` | Google Gemini |
| `GROQ_API_KEY` | Groq |
| `OPENROUTER_API_KEY` | OpenRouter |
| `AWS_ACCESS_KEY_ID` + `AWS_SECRET_ACCESS_KEY` + `AWS_REGION` | Amazon Bedrock |

## Quick Start Workflow

### Step 1: Initialize a Project

```bash
cd /path/to/your/project
crush
```

On first run, Crush prompts for an API key. Once configured, use `/init` to analyze the project
and create an `AGENTS.md` context file.

### Step 2: Install Oh-My-OpenCode-Slim Plugin

```bash
bunx oh-my-opencode-slim@latest install
```

Transforms Crush into a multi-agent system with specialized agents for different tasks.
See the [Crush Setup Guide](./opencode-setup.md) for full configuration details.

> **Air-Gap Warning**: `bunx` downloads from npm. For air-gapped environments, pre-install
> on a connected machine and transfer via internal npm mirror (Verdaccio).

### Step 3: Start Coding

```
crush
> Add a user authentication module with JWT tokens
```

### Step 4: Common Commands

| Command | Function |
|---------|----------|
| `/init` | Analyze project, generate AGENTS.md |
| `/undo` | Undo last modification |
| `crush logs` | Print recent logs |
| `crush logs --follow` | Follow logs in real time |
| `crush --yolo` | Skip all permission prompts (use with care) |

## Use Cases

### Use Case 1: New Feature Development

```
You: Add a caching layer to the API

Crush: Exploring existing API structure...
  Found: src/api/routes.py, src/services/

Plan:
  1. Create src/cache/ with Redis and in-memory adapters
  2. Add @cached decorator
  3. Update API endpoints

Proceed? [y/n]
```

### Use Case 2: Bug Investigation

```
You: Find where the login handler is defined and trace its execution flow

Crush: [Uses Serena tools to find symbol]
  Found login_handler in auth/routes.py:42

  Execution flow:
  - login_handler → authenticate_user (auth/service.py:15)
  - authenticate_user → validate_token (auth/jwt.py:8)
  - validate_token → decode_jwt (auth/jwt.py:22)
```

### Use Case 3: Refactoring

```
You: Rename UserService to AccountService and update all references

Crush: [Uses Serena's rename_symbol]
  Found 47 references across 12 files
  Updating...
  Updated 47 references in 12 files
```

## MCP Server Integration

### Disabling Specific MCP Tools

```json
{
  "mcp": {
    "serena": {
      "type": "stdio",
      "command": "uvx",
      "args": ["--from", "git+https://github.com/oraios/serena",
               "serena", "start-mcp-server", "--context", "ide-assistant", "--project", "."],
      "disabled_tools": ["execute_shell_command"]
    }
  }
}
```

### Disabling Built-In Tools

```json
{
  "options": {
    "disabled_tools": ["bash", "sourcegraph"]
  }
}
```

## Troubleshooting

### "crush: command not found"

```bash
# Verify installation
which crush

# Reinstall via Homebrew
brew install charmbracelet/tap/crush
```

### "MCP server timeout"

- Increase timeout in config: `"timeout": 120`
- Verify server is installed: `serena --version` or `srclight --version`
- For air-gapped: verify all dependencies are locally installed

### "Provider auto-update fails"

```bash
export CRUSH_DISABLE_PROVIDER_AUTO_UPDATE=1
```

### "Code not being indexed"

- Ensure `compile_commands.json` exists (for C/C++)
- Run `/init` to regenerate AGENTS.md

## Next Steps

- Read the [Architecture Overview](../architecture/overview.md)
- Set up [Crush with Oh-My-OpenCode-Slim](./opencode-setup.md) for multi-agent orchestration
- Configure [Serena](./serena-quickstart.md) for symbolic code navigation
- Configure [Srclight](./srclight-quickstart.md) for hybrid code search
- Review the [Security Assessment](../research/security-assessment.md) for air-gapped deployment