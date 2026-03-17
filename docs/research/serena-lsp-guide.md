# Serena LSP Tool: Best Practices for Air-Gapped C/C++ Codebase Intelligence

## Overview

**Serena** is an LSP-based coding agent toolkit that provides semantic code retrieval and editing capabilities at the symbol level. It is a core component of the air-gapped codebase intelligence architecture, working alongside **Srclight** (vector database), **Memora** (cross-session memory), and **OpenCode** (IDE interface).

> **Project Context:** This document is part of the air-gapped AI coding architecture, which implements a 100% offline RAG stack for C/C++ codebase intelligence.

---

## 1. Architecture Integration

Serena serves as the **symbolic navigation layer** in the project's hub-and-spoke architecture:

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode (Orchestration)                  │
│              (Hub-and-spoke agent coordination)              │
└─────────────────────────┬───────────────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌─────────┐      ┌──────────┐      ┌──────────┐
   │ Serena  │      │ Srclight │      │  Memora  │
   │  (LSP)  │      │ (Vector) │      │ (Memory) │
   └─────────┘      └──────────┘      └──────────┘
        │                 │                 │
        ▼                 ▼                 ▼
   Symbol-level     Semantic/Pattern   Historical
   Navigation       Search             Decisions
```

**Query Routing:**
- **Structural queries** (e.g., "find all calls to this function") → Serena
- **Semantic/pattern queries** (e.g., "find code similar to...") → Srclight
- **Historical/decision queries** (e.g., "why was this changed?") → Memora

---

## 2. Core Capabilities

| Capability | Description |
|------------|-------------|
| `find_symbol` | Locate functions, variables, or types by exact name |
| `find_referencing_symbols` | Discover all locations where a symbol is used |
| `insert_after_symbol` | Make precise code modifications respecting scope/indentation |
| `get_symbols_overview` | Get complete symbol graph of a file |

**Supported Languages:** 30+ including C/C++, Python, Rust, Go, Java, JavaScript, TypeScript, and more.

---

## 3. C/C++ Language Servers

Serena supports two C/C++ language server backends:

### 3.1 Clangd (Default - Recommended)

- **Official LLVM project** language server
- Built directly on clang compiler infrastructure
- **Automatic path transformation**: Converts relative paths in `compile_commands.json` to absolute paths
- Writes transformed config to `.serena/compile_commands.json`
- Better C++ standards compliance (C++17, C++20, C++23)
- Actively maintained with regular updates

### 3.2 CCLS (Alternative)

- Requires manual system installation
- Can handle relative paths natively (no transformation)
- Historically faster on very large codebases (50k+ files)
- Some unique features (call hierarchies, documentation handling)
- Less actively maintained than clangd

**Recommendation:** Use **clangd** (default) for most projects. CCLS is only recommended for teams with existing CCLS deployments or specific performance requirements on extremely large codebases.

---

## 4. Prerequisites & Setup

### 4.1 Compile Commands (Required)

**Non-negotiable:** A valid `compile_commands.json` file must exist at the repository root.

**Generation methods:**

```bash
# CMake (recommended)
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -B build .

# Bear (for other build systems)
bear -- make

# CMake with Ninja
cmake -GNinja -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -B build .
```

The file must include:
- Proper `-std=c++17` (or appropriate standard) flags
- Complete `-I` include paths
- All compiler definitions

### 4.2 Installation

```bash
# Recommended: via uv (fast, dependency-isolated)
uvx --from git+https://github.com/oraios/serena serena start-mcp-server --context ide-assistant --project "$(pwd)"

# Alternative: via pip
pip install serena

# Alternative: via uv pip
uv pip install serena

# For air-gapped: pre-install, then use locally
pip install serena  # on connected machine
# Transfer wheel files to air-gapped machine
```

> **Air-Gap Warning**: `npx -y @oraios/serena` always downloads from the npm registry.
> For air-gapped environments, pre-install via pip or npm: `npm install -g @oraios/serena`

### 4.3 Project Configuration

Create `.serena/project.yml` at the project root:

```yaml
# Default (clangd)
languages:
  - cpp

# Or use CCLS (requires manual installation)
languages:
  - cpp_ccls

# Custom compile commands directory (optional)
compile_commands_dir: "./custom_compile_commands"
```

---

## 5. Security Considerations

Serena provides powerful system access that requires careful configuration in air-gapped environments.

### 5.1 Known Vulnerabilities

A security audit (see [Discussion #380](https://github.com/oraios/serena/discussions/380)) identified:

| Finding | Severity | Description |
|---------|----------|-------------|
| Shell command execution | Critical | `ExecuteShellCommandTool` uses `subprocess.Popen(shell=True)` with no sandboxing |
| Network binding | High | Dashboard and MCP server bind to `0.0.0.0` by default (unauthenticated) |
| File system access | High | Path traversal possible via file_tools.py |

### 5.2 Hardening for Air-Gapped Deployment

```yaml
# .serena/project.yml - hardened configuration
read_only: true              # Disable file modifications unless needed
enable_gui_logging: false    # Disable dashboard (prevents 0.0.0.0 binding)
```

With this configuration, the `ide-assistant` context already excludes `execute_shell_command` by default, and the dashboard is fully disabled. In an air-gapped environment with a local LLM, this eliminates the critical attack vectors without requiring container isolation.

Additional measures:

- **Bind to localhost only**: If dashboard is needed, configure to `127.0.0.1`
- **Process privileges**: Run as non-root user with minimal permissions
- **Monitor connections**: Verify no unexpected outbound network activity
- **Container isolation (optional)**: Only warranted when `execute_shell_command` is enabled (for running tests/builds), when using a remote LLM provider, or when compliance mandates defense-in-depth. Container deployment adds complexity (volume mounts for project files, LSP binaries, and system headers).

See the full [Security Assessment](../research/security-assessment.md) for comprehensive analysis.

---

## 6. Air-Gapped Deployment

Serena is designed for **100% offline operation**. Code never leaves the local machine.

### 6.1 MCP Integration

Serena integrates with the **Model Context Protocol (MCP)** for local tool-model communication:

```
┌──────────────┐      IPC       ┌──────────────┐
│   Serena     │◄──────────────►│  OpenCode    │
│  MCP Server  │                │  (or Ollama) │
└──────────────┘                └──────────────┘
     │
     ▼
┌──────────────┐
│  Clangd/CCLS │  (all local)
│  (local)     │
└──────────────┘
```

### 6.2 OpenCode Integration

In this architecture, Serena integrates with **OpenCode** (the base AI coding agent):

```bash
# Configure OpenCode to use Serena MCP server
# In OpenCode settings or config file:

mcpServers:
  serena:
    command: serena
    args: ["--mcp"]
```

### 6.3 Prerequisites for Air-Gapped Use

Before going air-gapped, ensure:

1. **Language servers downloaded**: Clangd binaries for your platform
2. **Offline models**: Pre-download Ollama models (`ollama pull qwen3-embedding`)
3. **All pip/npm packages**: Install locally before disconnecting
4. **No cloud dependencies**: Verify no external API calls in configuration
5. **Environment variables**: Set `HF_HUB_OFFLINE=1` for any Python processes using Hugging Face
6. **NPX avoided**: Use pip-installed or npm-installed packages instead of `npx -y`

---

## 7. Best Practices

### ✅ Do

| Practice | Reason |
|----------|--------|
| Generate accurate `compile_commands.json` | Enables cross-file references and complete type analysis |
| Version control `.serena/project.yml` | Ensures team consistency |
| Restart language server after adding new files | New files aren't auto-indexed[6] |
| Configure timeouts for large codebases | Prevents failures on complex queries |
| Use clangd (default) | Better maintenance and standards compliance |

### ❌ Don't

| Practice | Reason |
|----------|--------|
| Skip `compile_commands.json` | Cross-file references won't work |
| Create files after server init without restart | New files won't be indexed |
| Use cloud-based LLM endpoints | Violates air-gapped requirement |
| Assume relative paths work with clangd | They're auto-transformed to absolute |

---

## 8. Known Limitations

| Limitation | Description | Workaround |
|------------|-------------|------------|
| **New files not indexed** | Files created after LSP init aren't auto-indexed | Restart language server or explicitly open new files |
| **Read-only paths** | LSP install fails on read-only directories | Use pre-installed system language servers |
| **Path transformation** | Clangd transforms relative paths | Verify `.serena/compile_commands.json` after setup |

---

## 9. Token Efficiency

Serena provides **~70% token savings** compared to text-based RAG approaches:

| Approach | Tokens for "find all calls to X" |
|----------|--------------------------------|
| Text-based RAG | ~5,000-10,000 (retrieve many files) |
| Serena (LSP) | ~500-1,000 (exact symbol graph) |

This efficiency is critical for:
- Staying within context window limits
- Reducing LLM costs (pay-per-token pricing)
- Faster response times

---

## 10. Integration with Srclight

Serena and **Srclight** (offline code indexer with hybrid search) complement each other:

| Query Type | Tool | Example |
|------------|------|---------|
| Structural | Serena | "Find all definitions of `process_data`" |
| Keyword | Srclight (FTS5) | "Find functions containing 'auth'" |
| Semantic | Srclight (embeddings) | "Find code that handles JSON parsing" |
| Hybrid | Srclight (FTS5 + semantic) | "Where is the authentication logic?" |
| Historical | Memora | "Why was this function modified?" |

**Combined workflow:**
1. Use Serena for precise symbol navigation
2. Use Srclight for hybrid (keyword + semantic) search across the codebase
3. Use Memora for cross-session context
4. OpenCode orchestrates all three

---

## 11. Quick Reference

```bash
# Generate compile_commands.json
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -B build .

# Install Serena
uv pip install serena

# Initialize project config
serena init

# Verify installation
serena doctor

# Restart language server (after adding files)
# Simply restart the MCP server or OpenCode session
```

---

## Related Documents

- [Architecture Overview](../architecture/overview.md)
- [Srclight Setup Guide](../guides/srclight-setup.md)
- [Memora Configuration](../guides/memora-config.md)
- [OpenCode Integration](../guides/opencode-setup.md)
- [Security Assessment](./security-assessment.md)

---

## References

- [Serena GitHub](https://github.com/oraios/serena)
- [Official Documentation](https://oraios.github.io/serena)
- [C/C++ Setup Guide](https://oraios.github.io/serena/03-special-guides/cpp_setup.html)
- [Security Audit Discussion](https://github.com/oraios/serena/discussions/380)
- [MCP Specification](https://modelcontextprotocol.io)