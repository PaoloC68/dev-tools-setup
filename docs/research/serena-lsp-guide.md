# Serena LSP Tool: Best Practices for Air-Gapped C/C++ Codebase Intelligence

## Overview

**Serena** is an LSP-based coding agent toolkit that provides semantic code retrieval and editing capabilities at the symbol level. It is a core component of the air-gapped codebase intelligence architecture, working alongside **Srclight** (vector database), **Memora** (cross-session memory), and **OpenCode** (IDE interface).

> **Project Context:** This document is part of the Oh-My-OpenCode-Slim project, which implements a 100% offline, air-gapped RAG stack for C/C++ codebase intelligence.

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
# Via pip
pip install serena

# Via uv (recommended)
uv pip install serena
```

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

## 5. Air-Gapped Deployment

Serena is designed for **100% offline operation**. Code never leaves the local machine.

### 5.1 MCP Integration

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

### 5.2 OpenCode Integration

For the Oh-My-OpenCode-Slim project, Serena integrates with **OpenCode** (the fork of the original archived tool):

```bash
# Configure OpenCode to use Serena MCP server
# In OpenCode settings or config file:

mcpServers:
  serena:
    command: serena
    args: ["--mcp"]
```

### 5.3 Prerequisites for Air-Gapped Use

Before going air-gapped, ensure:

1. **Language servers downloaded:** Clangd binaries for your platform
2. **Offline models:** Pre-download embedding models (e.g., `all-MiniLM-L6-v2`, `CodeT5+`, or Ollama-compatible)
3. **All pip/npm packages:** Install locally before disconnecting
4. **No cloud dependencies:** Verify no external API calls in configuration

---

## 6. Best Practices

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

## 7. Known Limitations

| Limitation | Description | Workaround |
|------------|-------------|------------|
| **New files not indexed** | Files created after LSP init aren't auto-indexed | Restart language server or explicitly open new files |
| **Read-only paths** | LSP install fails on read-only directories | Use pre-installed system language servers |
| **Path transformation** | Clangd transforms relative paths | Verify `.serena/compile_commands.json` after setup |

---

## 8. Token Efficiency

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

## 9. Integration with Srclight

Serena and **Srclight** (offline vector database) complement each other:

| Query Type | Tool | Example |
|------------|------|---------|
| Structural | Serena | "Find all definitions of `process_data`" |
| Semantic | Srclight | "Find code that handles JSON parsing" |
| Natural Language | Srclight | "Where is the authentication logic?" |
| Historical | Memora | "Why was this function modified?" |

**Combined workflow:**
1. Use Serena for precise symbol navigation
2. Use Srclight for semantic search across the codebase
3. Use Memora for cross-session context
4. OpenCode orchestrates all three

---

## 10. Quick Reference

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
- [Srclight Setup Guide](./srclight-setup.md)
- [Memora Configuration](./memora-config.md)
- [OpenCode Integration](./opencode-setup.md)

---

## References

- [Serena GitHub](https://github.com/oraios/serena)
- [Official Documentation](https://oraios.github.io/serena)
- [C/C++ Setup Guide](https://oraios.github.io/serena/03-special-guides/cpp_setup.html)
- [MCP Specification](https://modelcontextprotocol.io)
