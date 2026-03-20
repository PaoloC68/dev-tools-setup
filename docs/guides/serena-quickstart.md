# Serena Quickstart Guide

## What is Serena?

**Serena** (github.com/oraios/serena) is an open-source coding agent toolkit that provides IDE-like capabilities to any LLM through the Model Context Protocol (MCP). It uses the Language Server Protocol (LSP) to understand code at the **symbol level**, not just text.

### Key Features

- **Symbol-Level Understanding**: Knows where every function, class, and variable is defined
- **30+ Languages**: Python, TypeScript, JavaScript, Go, Rust, C/C++, Java, and more
- **100% Offline**: Works with local LLMs, no cloud dependency
- **IDE Integration**: Claude Desktop, VSCode, Cursor, OpenCode, IntelliJ, Cline
- **Token Efficient**: ~70% savings vs text-based RAG

## Installation

### Recommended: uvx (via uv)

```bash
uvx --from git+https://github.com/oraios/serena serena start-mcp-server --context ide-assistant --project "$(pwd)"
```

This pulls directly from the Serena repository and runs without a global install. Best option for reproducible, isolated environments.

### Alternative: pip

```bash
# Base install
pip install serena

# With MCP support
pip install "serena[mcp]"

# All languages (includes all LSP servers)
pip install "serena[all]"
```

### NPX

```bash
npx -y @oraios/serena --version
```

> **Air-Gap Warning**: `npx -y` always downloads from the npm registry.
> For air-gapped environments, pre-install globally: `npm install -g @oraios/serena`
> or use pip/uvx with pre-downloaded packages.

### Docker (Experimental)

```bash
docker run -it -v $(pwd):/workspace oraios/serena
```

## Configuration

### Claude Desktop

Add to `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "serena": {
      "command": "uvx",
      "args": [
        "--from", "git+https://github.com/oraios/serena",
        "serena", "start-mcp-server",
        "--context", "ide-assistant",
        "--project", "."
      ]
    }
  }
}
```

### Crush

Add to `.crush.json` (project root) or `~/.config/crush/crush.json` (global):

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
        "--context", "ide-assistant",
        "--project", "."
      ]
    }
  }
}
```

### Custom Project Path

```json
{
  "mcp": {
    "serena": {
      "type": "stdio",
      "command": "uvx",
      "args": [
        "--from", "git+https://github.com/oraios/serena",
        "serena", "start-mcp-server",
        "--context", "ide-assistant",
        "--project", "/path/to/your/project"
      ]
    }
  }
}
```

## Project Activation

### Onboarding Workflow

1. Install and start the Serena MCP server (see Installation above)
2. Connect a client (Claude Code, Crush, Claude Desktop, Cursor)
3. Prompt: `Activate the project at /path/to/project`
4. Serena analyzes structure, indexes symbols, builds dependency graph
5. Creates `.serena/project.yml` and `.serena/memories/`

### Configuration (.serena/project.yml)

```yaml
# Core settings
read_only: false
enable_gui_logging: true
onboarding_enabled: true

# Language support
languages:
  - python
  - cpp        # uses clangd by default
  # - cpp_ccls  # alternative: use ccls instead

# Exclusions (recommended for large codebases)
exclusions:
  - "**/node_modules/**"
  - "**/*build*"
  - "**/deps/**"
  - ".git"
```

## Verified Tools

### Navigation Tools

| Tool | Description |
|------|-------------|
| `find_symbol` | Locate symbols by exact name or partial match |
| `get_symbols_overview` | Get all symbols in a file |
| `find_referencing_symbols` | Find all symbols that REFERENCE a given symbol (callers) |
| `find_referenced_symbols` | Find all symbols REFERENCED BY a given symbol (callees) |

### Editing Tools

| Tool | Description |
|------|-------------|
| `replace_symbol_body` | Replace entire function/method body (preserves signature) |
| `insert_before_symbol` | Insert code before a symbol |
| `insert_after_symbol` | Insert code after a symbol |
| `delete_lines` | Delete specific lines |
| `rename_symbol` | Rename and update all references |

### Execution Tools

| Tool | Description |
|------|-------------|
| `execute_shell_command` | Run shell commands (test execution, build, VCS) |

### Meta Tools

| Tool | Description |
|------|-------------|
| `write_memory` | Persist insights to `.serena/memories/` |
| `read_memory` | Retrieve previously stored memories |
| `onboarding` | Automated project familiarization |
| `think_about_task_adherence` | Check task alignment |

## Use Cases

### Use Case 1: Complex Refactoring

**Scenario**: Rename a function across 50 files

```
You: Rename the function `authenticate_user` to `validate_credentials` and update all references

Serena:
  Found 47 references across 12 files
  - src/auth/routes.py:42
  - src/auth/service.py:15
  - tests/auth/test_service.py:8
  - ...

Updating references...
  Updated 47 references in 12 files
  Renamed symbol in 3 files

Note: Also updated docstrings and comments mentioning "authenticate_user"
```

### Use Case 2: Deep-Dive Debugging

**Scenario**: Trace a bug through complex call stack

```
You: The login is failing. Find the login handler and trace its execution flow

Serena:
Found `login_handler` in auth/routes.py:42

Execution flow:
1. login_handler (auth/routes.py:42)
   └─→ authenticate_user (auth/service.py:15)
       └─→ validate_token (auth/jwt.py:8)
           └─→ decode_jwt (auth/jwt.py:22)
               └─→ extract_claims (auth/jwt.py:31)

You: Show me the validate_token function

Serena:
```python
def validate_token(token: str) -> bool:
    try:
        claims = decode_jwt(token)
        return claims["exp"] > time.time()
    except ExpiredSignatureError:
        return False  # BUG: Should raise exception, not return False
```

You: The issue is the silent failure. Fix it to raise an exception instead.

Serena: [Uses replace_symbol_body to fix the function]
```

### Use Case 3: Understanding Large Codebases

**Scenario**: New team member exploring the codebase

```
You: I'm new to this project. Give me an overview of the architecture.

Serena:
## Project Structure Overview

### Core Modules
- `src/api/` - REST API endpoints (FastAPI)
- `src/services/` - Business logic layer
- `src/models/` - Database models (SQLAlchemy)
- `src/auth/` - Authentication (JWT)

### Key Entry Points
- `main.py:15` - FastAPI app initialization
- `src/api/routes.py:1` - All route handlers

### Data Flow
API → Services → Models → Database

### Dependencies
- External: PostgreSQL, Redis
- Internal: auth, models, services

Would you like me to dive into any specific module?
```

### Use Case 4: Precise Code Insertion

**Scenario**: Add logging to a specific function without breaking indentation

```
You: Add debug logging at the start of the `process_payment` function in payment.py

Serena:
Found `process_payment` in src/payment/processor.py:23

Inserting before symbol...

```python
def process_payment(amount: Decimal, user_id: int) -> PaymentResult:
    logger.debug(f"Processing payment: amount={amount}, user_id={user_id}")  # Added
    
    # Validate amount
    if amount <= 0:
        raise ValueError("Amount must be positive")
    ...
```

### Use Case 5: Memory and Context

**Scenario**: Remember important architectural decisions

```
You: Remember this: JWT tokens use 24-hour expiry and refresh tokens for session management.

Serena:
  Saved to .serena/memories/architecture.md

Content:
# Architecture Decision

## JWT Token Strategy
- Access tokens: 24-hour expiry
- Refresh tokens: 7-day expiry
- Used for session management across all API endpoints

Last updated: 2024-01-15
```

You: What did the team decide about token expiry?

Serena:
From memory (`.serena/memories/architecture.md`):
- Access tokens: 24-hour expiry
- Refresh tokens: 7-day expiry
```

## Language Support

### Direct Support (Best Experience)

| Language | LSP Server | Notes |
|----------|------------|-------|
| Python | pylsp / pyright | Best |
| TypeScript/JavaScript | typescript-language-server | Best |
| Go | gopls | Best |
| Rust | rust-analyzer | Best |
| C/C++ | clangd / ccls | Best |
| Java | eclipse.jdtls | Good |
| PHP | php-language-server | Good |

### Indirect Support (via multilspy)

Ruby, C#, Kotlin, Dart, Scala, Swift, and more.

### C/C++ Requirements

For C/C++ projects, generate `compile_commands.json`:

```bash
# CMake
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..

# Make (with bear)
bear -- make
```

## Security Considerations

Serena provides powerful system access. The `ide-assistant` and `claude-code` contexts already exclude the most dangerous tool (`execute_shell_command`) by default.

**Recommended hardening** (sufficient for air-gapped + local LLM environments):

```yaml
# .serena/project.yml
enable_gui_logging: false           # Disables dashboard (no 0.0.0.0 binding)

excluded_tools:
  - execute_shell_command           # Explicitly remove shell access
```

This eliminates the two critical risks: arbitrary shell execution and unauthenticated network dashboard. File access remains, but with an air-gapped network and local LLM, there is no exfiltration path.

**Container isolation is optional** and only warranted when:
- Using a remote LLM provider (cloud API — file contents travel to cloud)
- Enabling `execute_shell_command` (for running tests/builds through Serena)
- Compliance requires defense-in-depth regardless of threat model

See the [Security Assessment](../research/security-assessment.md) for detailed analysis and container setup instructions when needed.

## Troubleshooting

### "Language server not found"

```bash
# Install language servers
pip install python-lsp-server  # Python
npm install -g typescript-language-server  # TypeScript
cargo install rust-analyzer  # Rust
```

### "Project not activated"

```
Activate the project at /absolute/path/to/your/project
```

### "Symbols not found"

- Ensure language server is running
- Check `compile_commands.json` exists (C/C++)
- Run onboarding again: `onboarding`

### Slow startup (Java)

- Java LSP can be slow on macOS
- Consider using pyright for Python-only projects

## Best Practices

- **Restart the language server** after adding new files. Newly created files aren't auto-indexed.
- **Choose the right context mode**: use `agent` for autonomous workflows, `ide-assistant` for interactive sessions.
- **Large C++ codebases** (>1M LOC): consider ccls over clangd for faster indexing.
- **Pre-index nightly** for large repos to avoid cold-start delays.
- **Version control `.serena/project.yml`** for team consistency. Keep `.serena/memories/` in `.gitignore`.

## Next Steps

- Read the [Architecture Overview](../architecture/overview.md)
- Set up [Crush](./opencode-quickstart.md) for orchestration
- Configure [Srclight](./srclight-quickstart.md) for semantic search