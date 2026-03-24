# OpenCode Quickstart Guide

## What is OpenCode?

**OpenCode** (opencode.ai) is an open-source AI coding agent with 120k+ GitHub stars. It provides a terminal-based interface, desktop app, or IDE extension for AI-assisted programming.

### Key Features
- **100% Open Source** — MIT License, fully customizable
- **Any Model** — 75+ LLM providers including Claude, GPT, Gemini, and local models
- **Privacy First** — No code storage, supports local execution
- **MCP Support** — Add custom MCP servers for extended capabilities
- **Multi-session** — Start multiple agents in parallel on the same project

## Installation

### Quick Install (Recommended)
```bash
curl -fsSL https://opencode.ai/install | bash
```

### Platform-Specific
```bash
# Node.js (npm/bun/pnpm/yarn)
npm i -g opencode-ai@latest

# Arch Linux
sudo pacman -S opencode        # stable
paru -S opencode-bin           # latest from AUR

# Debian / Ubuntu
npm i -g opencode-ai@latest    # recommended
# or grab the .deb from https://github.com/anomalyco/opencode/releases

# Windows with Scoop
scoop install opencode

# Windows with Chocolatey
choco install opencode
```

## Configuration

### Environment Variables

```bash
export INFERENCE_API_KEY="your-internal-key"
```

This is the only key needed. No cloud provider keys (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, etc.)
are used — all model access goes through the internal OpenAI-compatible inference server.

The `{env:INFERENCE_API_KEY}` syntax in `opencode.json` reads this variable at runtime.

### Configuration File (opencode.json)

Place in project root, or at `~/.config/opencode/opencode.json` for global settings:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "autoupdate": false,
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
  "model": "internal/Qwen3.5-397B-A17B",
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
      "command": ["srclight", "serve"],
      "enabled": true,
      "environment": {
        "OPENAI_API_KEY": "{env:INFERENCE_API_KEY}"
      }
    },
    // Note: indexing is done via CLI before starting the server:
    //   OPENAI_API_KEY=sk-xxx srclight index \
    //     OPENAI_BASE_URL=http://inference.internal srclight index --embed openai:qwen3-embedding-8b
    "memora": {
      "type": "local",
      "command": ["memora-server"],
      "enabled": true,
      "environment": {
        "MEMORA_DB_PATH": "~/.local/share/memora/memories.db",
        "MEMORA_EMBEDDING_MODEL": "sentence-transformers",
        "MEMORA_LLM_ENABLED": "false",
        "MEMORA_ALLOW_ANY_TAG": "1"
      }
    }
  }
}
```

> **Note on Srclight embeddings**: `OPENAI_API_KEY` is passed to the running server so the
> `reindex()` MCP tool can re-embed. Initial indexing uses the `--embed` URL flag:
> ```bash
> OPENAI_API_KEY=your-key OPENAI_BASE_URL=http://inference.internal srclight index \
>   --embed openai:qwen3-embedding-8b
> ```

> **Air-Gap Note**: Replace `git+https://` with locally installed packages.
> Pre-install Serena via `pip install "serena[mcp]"` and use the local binary path.

## Quick Start Workflow

### Step 1: Initialize a Project
```bash
cd /path/to/your/project
opencode

# In OpenCode terminal
/init
```

This creates `AGENTS.md`, an AI-friendly project description.

### Step 2: Install Oh-My-OpenCode-Slim Plugin

```bash
bunx oh-my-opencode-slim@latest install
```

This transforms OpenCode into a multi-agent system with specialized agents for different tasks. See the [OpenCode Setup Guide](./opencode-setup.md#oh-my-opencode-slim-plugin) for full configuration details.

> **Air-Gap Warning**: `bunx` downloads from npm. For air-gapped environments, pre-install
> on a connected machine and transfer via internal npm mirror (Verdaccio).

### Step 3: Start Coding

#### Plan Mode (Review Before Building)
```
Press Tab to switch to Plan mode
> Add a user authentication module with JWT tokens

# Review the plan, then press Tab to switch to Build mode
```

#### Build Mode (Direct Changes)
```
Press Tab to switch to Build mode
> Create a function that validates email addresses
```

### Step 4: Common Commands

| Command | Function |
|---------|----------|
| `/init` | Initialize project, generate AGENTS.md |
| `/connect` | Connect or switch AI provider |
| `/undo` | Undo last modification |
| `/redo` | Redo previously undone modification |
| `/share` | Generate shareable link to conversation |
| `/clear` | Clear current session |

## Use Cases

### Use Case 1: New Feature Development
```
You: I need to add a caching layer to our API

OpenCode: [Switches to Plan mode]

Plan:
1. Create cache/ directory with __init__.py
2. Implement CacheManager class with TTL support
3. Add Redis adapter for production
4. Create in-memory adapter for development
5. Update API endpoints to use caching

You: Looks good, but use memcached instead of Redis

OpenCode: Updated plan to use memcached...

You: Ready to build

OpenCode: [Switches to Build mode, implements changes]
```

### Use Case 2: Bug Investigation
```
You: Find where the login handler is defined and trace its execution flow

OpenCode: [Uses Serena tools to find symbol]
Found login_handler in auth/routes.py:42

Tracing flow:
- login_handler → authenticate_user (auth/service.py:15)
- authenticate_user → validate_token (auth/jwt.py:8)
- validate_token → decode_jwt (auth/jwt.py:22)

You: The token validation seems to be the issue. Show me the validate_token function

OpenCode: [Shows the function code]
```

### Use Case 3: Code Review
```
You: Review the payment processing code for security issues

OpenCode: Analyzing payment processing module...

Findings:
1. [HIGH] No input validation on amount field - potential float injection
2. [MEDIUM] Missing rate limiting on payment endpoint
3. [LOW] Hardcoded API key found in test file

Would you like me to fix these issues?
```

### Use Case 4: Refactoring
```
You: Rename the UserService class to AccountService and update all references

OpenCode: [Uses Serena's native refactoring]

Found 47 references across 12 files:
- src/models/user.py
- src/services/user.py
- src/api/routes.py
- tests/

Updating...
✓ Updated 47 references in 12 files
```

## MCP Server Integration

### Adding Local MCP Servers
```json
{
  "$schema": "https://opencode.ai/config.json",
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

> **Air-Gap Warning**: `uvx` with `git+https://` downloads at runtime.
> For air-gapped environments, pre-install Serena via `pip install serena` and use the local binary path.

### Per-Agent MCP Configuration
```json
{
  "$schema": "https://opencode.ai/config.json",
  "mcp": {
    "serena": {
      "type": "local",
      "command": ["uvx", "--from", "git+https://github.com/oraios/serena",
                  "serena", "start-mcp-server", "--context", "ide-assistant", "--project", "."]
    }
  },
  "tools": {
    "serena_*": false
  },
  "agent": {
    "reviewer": {
      "tools": {
        "serena_*": true
      }
    }
  }
}
```

## Troubleshooting

### "Model not found"
- Verify provider and model naming format
- Check API key is valid and has credits

### "MCP server timeout"
- Increase timeout in config: `"timeout": 30000`
- Check server is installed: `serena --version` or `srclight --version`
- For air-gapped: verify all dependencies are locally installed

### "Code not being indexed"
- Ensure `compile_commands.json` exists (for C/C++)
- Run `/init` to regenerate AGENTS.md

## Next Steps

- Read the [Architecture Overview](../architecture/overview.md)
- Set up [Serena](./serena-quickstart.md) for symbolic code navigation
- Configure [Srclight](./srclight-quickstart.md) for hybrid code search
- Review the [Security Assessment](../research/security-assessment.md) for air-gapped deployment