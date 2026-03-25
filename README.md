# Air-Gapped AI-Assisted Coding Architecture

100% offline RAG stack for C/C++ codebase intelligence using the Model Context Protocol (MCP).

## The Problem

AI coding assistants are powerful but unreliable at scale. Four failure modes dominate:

**Hallucinations.** The LLM invents functions that don't exist, imports nonexistent modules, or rewrites code that already exists elsewhere in the codebase. Without structural awareness of what's actually in the repository, the model guesses — and guesses wrong. The larger the codebase, the worse this gets.

**Context rot.** As the conversation grows, the LLM loses grip on earlier information. Research shows quality degrades sharply past ~32k tokens (Liu et al., 2023). Architecture decisions made at the start of a session get forgotten by the middle. The agent contradicts its own plan, or silently ignores project conventions it was told about 20 messages ago.

**Memory loss across sessions.** Every new session starts from zero. The agent re-explores the same files, re-discovers the same patterns, re-makes the same decisions — sometimes differently. Weeks of accumulated project understanding vanish when the context window resets.

**Convention blindness.** The LLM writes functionally correct code that ignores existing project patterns — naming conventions, error handling style, import structure, architectural idioms. The larger the codebase, the worse this gets: the agent has never "seen" most of it, so it invents from scratch instead of matching what exists.

### How This Architecture Addresses Each

| Problem | Root Cause | Solution in This Stack |
|---------|-----------|----------------------|
| **Hallucinations** | LLM lacks structural knowledge of the codebase | **Serena** provides symbol-level code intelligence via LSP — the agent queries actual definitions, references, and callers instead of guessing. **Srclight** adds hybrid search so the agent finds existing code before writing new code. |
| **Context rot** | Single context window overloaded with planning + code + history | **Oh-My-OpenCode-Slim** isolates each agent in its own context window (8-30k tokens each vs 120k+ shared). The Explorer searches in one context, the Coder writes in another — neither pollutes the other. |
| **Memory loss** | No persistence between sessions | **Memora** stores decisions, insights, and architectural context in a local SQLite database with hybrid search. **Serena memories** persist project-specific knowledge as markdown files. The agent picks up where it left off. |
| **Convention blindness** | LLM has never seen most of the codebase | **Srclight** finds existing implementations that match the task, so the agent follows established patterns. **Serena** exposes the actual code structure (callers, callees, type hierarchies), not just file contents. **Memora** persists project conventions across sessions — discovered once, applied always. |

The net effect: the LLM operates with **verified structural context** (not guesses), **focused per-task windows** (not an overloaded single conversation), **persistent cross-session memory** (not a blank slate every time), and **awareness of existing patterns** (not isolated invention).

For the detailed analysis, see [Why Orchestration](docs/research/orchestration-vs-chat.md).

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenCode (Base Agent)                     │
│              Terminal AI Coding Assistant                    │
├─────────────────────────────────────────────────────────────┤
│              Oh-My-OpenCode-Slim (Plugin)                   │
│         Multi-Agent Orchestration & Task Delegation          │
│                                                             │
│  Sisyphus   Prometheus   Oracle   Explorer                  │
│  Librarian  Designer     Fixer                              │
└─────────────────────────┬───────────────────────────────────┘
                          │ MCP (stdio)
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
    ┌──────────┐   ┌──────────┐   ┌──────────┐
    │  Serena  │   │ Srclight │   │  Memora  │
    │(Symbolic)│   │(Semantic)│   │(Memory)  │
    └──────────┘   └──────────┘   └──────────┘
       clangd       tree-sitter    SQLite
       ccls         FTS5 + embeddings
                    (internal server)
```

## Components

| Component | Role | What It Does |
|-----------|------|-------------|
| **[OpenCode](https://opencode.ai)** | Base Agent | Terminal AI coding assistant. Plugin system, MCP support, multi-provider. |
| **[Oh-My-OpenCode-Slim](https://github.com/alvinunreal/oh-my-opencode-slim)** | Orchestration Plugin | Transforms OpenCode into a multi-agent system. 7 specialized agents, model routing, background tasks, lifecycle hooks. |
| **[Serena](docs/guides/serena-quickstart.md)** | Symbolic MCP Server | LSP-based code navigation via clangd/ccls. 30+ languages, 13 tools. |
| **[Srclight](docs/guides/srclight-quickstart.md)** | Semantic MCP Server | Hybrid search: FTS5 trigram + embeddings via RRF. 29 tools, 11 languages. |
| **[Memora](docs/guides/memora-config.md)** | Persistence MCP Server | Cross-session context, SQLite backend. Cloud sync DISABLED for air-gap. |

## Why Oh-My-OpenCode-Slim

Without this plugin, OpenCode runs as a single agent. With it:

- **7 specialized agents** handle different task types in parallel
- **Model routing** assigns cheap models to exploration, expensive models to planning
- **Background tasks** run concurrently (Explorer, Librarian)
- **MCP coordination** routes queries to Serena, Srclight, or Memora automatically
- **25+ lifecycle hooks** manage session behavior and compaction

| Agent | Role | Typical Task |
|-------|------|-------------|
| Sisyphus | Orchestrator | Delegates to specialists, tracks progress via TODOs |
| Prometheus | Planner | Creates parallel task graphs before complex implementations |
| Oracle | Advisor | Architecture review, debugging after failed attempts |
| Explorer | Recon | Parallel codebase search, runs in background |
| Librarian | Knowledge | External docs, OSS examples, GitHub search |
| Designer | Visual | Frontend, UI/UX, styling |
| Fixer | Implementation | Bug fixes, single-file changes |

## Quick Start

```bash
# 1. Install OpenCode
curl -fsSL https://opencode.ai/install | bash
# or: npm i -g opencode-ai@latest

# 2. Install Oh-My-OpenCode-Slim plugin
bunx oh-my-opencode-slim@latest install

# 3. Install MCP servers
# Note: download these on a connected machine and transfer wheels for air-gap
pip install "serena[mcp]"
# tree-sitter-dart is not on PyPI — use --no-deps + manual dep list (see srclight-quickstart.md)
pip download "srclight==0.15.1" --no-deps -d ./wheels/
pip install --no-index --find-links ./wheels/ --no-deps srclight==0.15.1
pip install "memora-mcp[local] @ git+https://github.com/agentic-box/memora.git"

# 4. Verify internal inference server is reachable
curl http://inference.internal/v1/models

# 5. Generate compile_commands.json (C/C++ projects)
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -B build .

# 6. Index your codebase
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal srclight index \
  --embed openai:qwen3-embedding-8b

# 7. Start MCP servers and launch
srclight serve --workspace myworkspace &
opencode
```

> **Air-Gap Note**: Download all packages, models, and dependencies BEFORE disconnecting.
> `bunx` and `uvx git+https://` always download — pre-install everything before going offline.

## Documentation Index

### Architecture

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture/overview.md) | Component hierarchy, agent system, SOTA comparison, scalability, centralized management |

### Guides

| Document | Description |
|----------|-------------|
| [OpenCode Quickstart](docs/guides/opencode-quickstart.md) | Install OpenCode and Oh-My-OpenCode-Slim plugin |
| [OpenCode Setup](docs/guides/opencode-setup.md) | Plugin config, agent model routing, enterprise deployment |
| [Serena Quickstart](docs/guides/serena-quickstart.md) | LSP-based symbolic navigation, 13 tools, C/C++ setup |
| [Srclight Quickstart](docs/guides/srclight-quickstart.md) | Hybrid code search, 29 MCP tools, tree-sitter, 11 languages |
| [Srclight Setup](docs/guides/srclight-setup.md) | Installation, embedding config, multi-repo workspaces |
| [Memora Quickstart](docs/guides/memora-quickstart.md) | Persistence layer setup, air-gap compliance, tools reference |
| [Memora Configuration](docs/guides/memora-config.md) | Persistence layer configuration reference |
| [Typical Workflow](docs/guides/typical-workflow.md) | Day-in-the-life: clone, index, search, refactor, commit, switch projects |

### Research

| Document | Description |
|----------|-------------|
| [Security Assessment](docs/research/security-assessment.md) | Threat model, 10 findings (1 CRITICAL), 16-item air-gap checklist, hardening |
| [OMO Slim Air-Gap Analysis](docs/research/omo-slim-airgap-analysis.md) | Network dependency audit, MCP disabling, binary pre-caching, hook security |
| [Why Orchestration](docs/research/orchestration-vs-chat.md) | Multi-agent vs single-agent analysis: context rot, model routing, RAG layers |
| [Comparative Analysis](docs/research/comparative-analysis.md) | Layer-by-layer evaluation of alternatives for each architecture component |
| [Model-to-Agent Mapping](docs/research/model-agent-mapping.md) | Optimal local model assignment per OMO Slim agent (7 models analyzed) |
| [Serena LSP Best Practices](docs/research/serena-lsp-guide.md) | clangd vs ccls, token efficiency, security hardening |

## Key Technical Details

- **Orchestration**: Oh-My-OpenCode-Slim plugin with 7 agents, model routing, background tasks
- **Embeddings**: Internal OpenAI-compatible server (`qwen3-embedding-8b`)
- **Search**: Hybrid — FTS5 trigram + semantic via Reciprocal Rank Fusion
- **Parsing**: tree-sitter (C, C++, Python, TypeScript, JavaScript, Rust, Go)
- **C/C++ LSP**: clangd (recommended) or ccls (large codebases)
- **Storage**: SQLite everywhere — no external databases
- **MCP Latency**: ~50-200ms per call
- **Token Savings**: ~70% vs text-based RAG

## Security

Air-gapped by design. The CRITICAL and HIGH labels below refer to known risks in default configurations that this architecture explicitly mitigates — they are documented precisely because they are handled.

| Risk | Severity | Mitigation |
|------|----------|------------|
| Serena shell execution | CRITICAL | Excluded by default in `ide-assistant` context; add to `excluded_tools` in project.yml |
| Serena binds to 0.0.0.0 | HIGH | Set `enable_gui_logging: false` in project.yml |
| MCP prompt injection | HIGH | Pin server versions, audit tool descriptions |
| Memora cloud sync | MEDIUM | Leave `MEMORA_STORAGE_URI` unset |
| Model access | MEDIUM | Verify internal inference server is reachable before going air-gap |
| OMO Slim bunx install | MEDIUM | Pre-install via npm cache or Verdaccio mirror |

Full analysis: [Security Assessment](docs/research/security-assessment.md)

## Repository Structure

```
docs/
├── architecture/
│   └── overview.md              # System design, agents, SOTA comparison
├── guides/
│   ├── opencode-quickstart.md   # Base agent + plugin setup
│   ├── opencode-setup.md        # Plugin config, enterprise deployment
│   ├── serena-quickstart.md     # Symbolic layer (LSP)
│   ├── srclight-quickstart.md   # Semantic layer (hybrid search)
│   ├── srclight-setup.md        # Srclight installation and config
│   └── memora-config.md         # Persistence layer
└── research/
    ├── security-assessment.md   # Full security audit
    └── serena-lsp-guide.md      # LSP best practices
```

This repository contains only documentation. No source code, no build system, no tests.

## License

See individual component licenses: [OpenCode](https://github.com/anomalyco/opencode) (MIT), [Oh-My-OpenCode-Slim](https://github.com/alvinunreal/oh-my-opencode-slim) (MIT), [Serena](https://github.com/oraios/serena) (MIT), [Srclight](https://github.com/srclight/srclight) (MIT), [Memora](https://github.com/agentic-mcp-tools/memora) (MIT).
