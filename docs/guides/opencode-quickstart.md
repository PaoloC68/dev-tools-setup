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
# macOS/Linux with Homebrew
brew install opencode-ai/tap/opencode

# Windows with Scoop
scoop install opencode

# Windows with Chocolatey
choco install opencode

# Node.js
npm i -g opencode-ai@latest
```

## Configuration

### Connect an LLM Provider
```bash
# Run in OpenCode terminal
/connect
```

Select your provider and enter API key. Recommended: OpenCode Zen (curated models).

### Environment Variables
```bash
export OPENAI_API_KEY="your-api-key"
export OPENAI_API_BASE="https://api.openai.com/v1"

# Or for local models (Ollama)
export OLLAMA_HOST="http://localhost:11434"
```

### Configuration File (~/.opencode.json)
```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "claude-sonnet-4-5",
  "provider": "openai",
  "mcp": {
    "serena": {
      "type": "local",
      "command": ["npx", "-y", "@oraios/serena"],
      "enabled": true
    }
  }
}
```

## Quick Start Workflow

### Step 1: Initialize a Project
```bash
cd /path/to/your/project
opencode

# In OpenCode terminal
/init
```

This creates `AGENTS.md` — an AI-friendly project description.

### Step 2: Start Coding

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

### Step 3: Common Commands

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
  "mcp": {
    "serena": {
      "type": "local",
      "command": ["npx", "-y", "@oraios/serena"],
      "enabled": true
    },
    "filesystem": {
      "type": "local",
      "command": ["npx", "-y", "@modelcontextprotocol/server-filesystem", "/path/to/allowed"],
      "enabled": true
    }
  }
}
```

### Adding Remote MCP Servers
```json
{
  "mcp": {
    "sentry": {
      "type": "remote",
      "url": "https://mcp.sentry.dev/mcp",
      "enabled": true
    },
    "gh_grep": {
      "type": "remote",
      "url": "https://mcp.grep.app",
      "enabled": true
    }
  }
}
```

### Per-Agent MCP Configuration
```json
{
  "agent": {
    "reviewer": {
      "tools": {
        "serena*": true,
        "gh_grep*": true
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
- Check server is installed: `npx -y @oraios/serena --version`

### "Code not being indexed"
- Ensure `compile_commands.json` exists (for C/C++)
- Run `/init` to regenerate AGENTS.md

## Next Steps

- Read the [Architecture Overview](../architecture/overview.md)
- Set up [Serena](./serena-quickstart.md) for symbolic code navigation
- Configure [Local Embeddings](./srclight-setup.md) for semantic search
