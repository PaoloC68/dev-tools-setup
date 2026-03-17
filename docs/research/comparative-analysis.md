# Comparative Analysis: Air-Gapped AI Coding Architecture Alternatives

## Evaluation Criteria

Every alternative is evaluated against four hard requirements:

| Requirement | Definition | Pass/Fail |
|-------------|-----------|-----------|
| **Air-gapped** | Zero runtime network access after initial setup | Binary |
| **Large codebase** | Handles C/C++ repos with >100k files | Binary |
| **Fully open source** | OSI-approved license, full source available | Binary |
| **Auditable** | Codebase is inspectable, no obfuscated components | Binary |

Soft criteria (scored 1-5): community size, maturity, token efficiency, integration quality, resource requirements.

---

## Layer 1: Base AI Coding Agent

The base agent provides the terminal interface, LLM communication, tool execution, and MCP server coordination.

### Current Choice: OpenCode

| Attribute | Value |
|-----------|-------|
| License | MIT |
| Stars | 124k |
| Language | Go |
| MCP Support | Native (first-class) |
| Plugin System | Yes (npm-based, hooks, custom tools) |
| Local LLM | Yes (Ollama, vLLM, any OpenAI-compatible endpoint) |
| Air-gap Install | Yes (`curl` binary or npm, no runtime downloads) |
| C/C++ | General (via MCP servers) |

### Alternatives

| Agent | License | MCP | Plugin System | Multi-Agent | Air-Gap Ready | Stars | Verdict |
|-------|---------|-----|---------------|-------------|---------------|-------|---------|
| **OpenCode** | MIT | Native | Yes (npm) | Via plugin (OMO) | Yes | 124k | **Current choice** |
| **Aider** | AGPL-3.0 | No | No | No | Partial | ~20k | AGPL problematic for enterprise; no MCP |
| **Continue.dev** | Apache-2.0 | Partial | Yes (IDE ext) | No | Partial (IDE dependency) | ~20k | IDE-locked (VSCode/JetBrains); no terminal |
| **Tabby** | Apache-2.0 | No | Limited | No | Yes (Docker) | ~25k | Completion-focused, not agentic; no MCP |
| **Cody (Sourcegraph)** | Apache-2.0 | No | IDE ext | No | Partial (graph indexing) | ~10k | Heavy infrastructure; not MCP-native |
| **Goose** | MIT | Yes | Limited | No | Partial | ~15k | Newer, less mature; single-agent |
| **Roo Code** | MIT | Yes | VSCode ext | No | Partial | ~15k | IDE-locked; no terminal mode |
| **Cline** | Apache-2.0 | Yes | VSCode ext | No | Partial | ~20k | IDE-locked; strong but not standalone |

### Analysis

**OpenCode is the strongest choice** for this architecture because:

1. **MCP is native** — not bolted on. OpenCode was designed around MCP from the start. Alternatives (Aider, Tabby, Continue) either lack MCP entirely or add it as an afterthought.
2. **Plugin system enables orchestration** — the OMO Slim plugin transforms single-agent into multi-agent. No other agent has a comparable plugin ecosystem for orchestration.
3. **Terminal-first** — works in any environment (SSH, containers, CI/CD). IDE-locked tools (Continue, Roo, Cline) require a desktop environment.
4. **MIT license** — no AGPL (Aider) or enterprise restrictions.
5. **Largest community** — 124k stars provides ecosystem momentum.

**Closest alternative**: Goose (MIT, MCP support, CLI) but lacks plugin system and multi-agent capabilities. Would require building orchestration from scratch.

**Disqualified alternatives**:
- Aider: AGPL-3.0 creates distribution concerns for enterprise/defense; no MCP support
- Tabby: Completion-focused, not an agentic coding assistant
- IDE-locked tools (Continue, Roo, Cline): Cannot operate in headless/containerized air-gapped environments

---

## Layer 2: Multi-Agent Orchestration

The orchestration layer transforms a single agent into a coordinated multi-agent system with specialized roles.

### Current Choice: Oh-My-OpenCode-Slim

| Attribute | Value |
|-----------|-------|
| License | MIT |
| Stars | 2.2k |
| Agents | 7 (Sisyphus, Prometheus, Oracle, Explorer, Librarian, Designer, Fixer) |
| Model Routing | Yes (per-agent model assignment) |
| Background Tasks | Yes (parallel agents) |
| Air-gap Config | `disabled_mcps` field, local model routing |

### Alternatives

| Plugin | License | Agents | Model Routing | Token Efficiency | Air-Gap Ready | Verdict |
|--------|---------|--------|---------------|------------------|---------------|---------|
| **OMO Slim** | MIT | 7 | Yes | High (~10k base) | Yes (with config) | **Current choice** |
| **Oh-My-OpenCode** (original) | Other* | 7+ | Yes | Lower (~25k base) | Similar | Higher token usage; OMO Slim is the optimized fork |
| **OpenAgentsControl** | OSS | Custom | Limited | Very high (80% reduction claim) | Similar | Editable prompts, but limited parallelization |
| **Superpowers** | Unknown | Skills-based | No | Medium | Unknown | Claude Code only; not OpenCode compatible |
| **No orchestration** (vanilla OpenCode) | N/A | 1 | No | Lowest overhead | Best (no plugin) | Single-agent limitations at scale (see orchestration-vs-chat.md) |

*Oh-My-OpenCode has changed its license and rebranded to oh-my-openagent.

### Analysis

**OMO Slim is the strongest choice** because:

1. **Token-optimized fork** — ~10k base tokens vs ~25k for the original Oh-My-OpenCode. Critical for local models with smaller context windows.
2. **`disabled_mcps` config** — the only orchestration plugin with a built-in mechanism to disable internet-dependent MCPs.
3. **Active maintenance** — 2.2k stars, 30 contributors, 13 releases, last updated March 2026.
4. **Mutual exclusivity** enforced — cannot install alongside the original OMO (prevents conflicts).

**Closest alternative**: OpenAgentsControl offers editable agent prompts and claims 80% token reduction, but lacks OMO Slim's parallel agent execution and extensive feature set (tmux, ast-grep, skills system).

**Consideration**: For maximum air-gap simplicity, running vanilla OpenCode (no orchestration) eliminates the plugin's network dependencies entirely. This trades multi-agent benefits for zero configuration overhead. Viable for smaller codebases or simpler tasks.

---

## Layer 3: Symbolic Code Intelligence (LSP)

The symbolic layer provides precise, structure-aware code navigation — finding definitions, references, callers, callees at the symbol level.

### Current Choice: Serena

| Attribute | Value |
|-----------|-------|
| License | MIT |
| Stars | 3.6k |
| Languages | 30+ (via LSP ecosystem) |
| MCP Server | Yes (native) |
| C/C++ Backend | clangd (default), ccls (alternative) |
| Tools | 13 (find_symbol, replace_symbol_body, rename_symbol, etc.) |
| Air-gap | Fully offline (pip install, no runtime downloads) |

### Alternatives

| Tool | License | MCP Native | Languages | C/C++ Quality | Air-Gap | Resource Usage | Verdict |
|------|---------|------------|-----------|---------------|---------|----------------|---------|
| **Serena** | MIT | Yes | 30+ | Excellent (clangd/ccls) | Yes | Medium (LSP RAM) | **Current choice** |
| **clangd-mcp-server** | MPL-2.0 | Yes | C/C++ only | Excellent | Yes | Low | Strong for C/C++ only projects |
| **Direct LSP (no wrapper)** | N/A | No | Any | Excellent | Yes | Low | Requires custom MCP adapter per project |
| **tree-sitter** (standalone) | MIT | No | 100+ | Good (structural, no types) | Yes | Very low | No type info, no cross-file references |
| **Sourcegraph (self-hosted)** | Apache-2.0 | Partial | Many | Good (SCIP/LSIF) | Yes | High (4-16GB) | Overkill for single-developer; enterprise-grade |
| **ctags/cscope** | GPL/BSD | No | C/C++ focused | Basic | Yes | Very low | No semantic understanding; symbol-level only |
| **CodeGraph RAG** | Community | Yes | Multiple | Unknown | Unknown | Unknown | Too new, insufficient track record |

### Analysis

**Serena is the strongest choice** because:

1. **Only mature MCP-native LSP wrapper** — the clangd-mcp-server alternative is C/C++ only, while Serena supports 30+ languages through a unified interface.
2. **Symbol-level editing** — `replace_symbol_body`, `insert_before_symbol`, etc. operate at the semantic level, not line numbers. This is unique to Serena among MCP servers.
3. **Onboarding system** — automated project familiarization that builds a symbol graph on first connection.
4. **Memory integration** — `.serena/memories/` provides file-based persistence without external dependencies.

**Closest alternative**: `clangd-mcp-server` (MPL-2.0) provides 9 code intelligence tools directly wrapping clangd. For C/C++-only projects, this is lighter than Serena and has fewer security concerns (no shell execution, no dashboard binding). **Consider as a hardened alternative** if Serena's security vulnerabilities (see security-assessment.md) are unacceptable.

**Not recommended**:
- Direct LSP without MCP wrapper: Requires building custom MCP adapters, reinventing what Serena already provides
- ctags/cscope: No semantic understanding, no type information, no cross-file reference resolution
- Sourcegraph: Designed for enterprise teams with dedicated infrastructure; excessive for single-developer air-gapped setups

---

## Layer 4: Semantic Code Search (Embeddings + RAG)

The semantic layer enables natural-language code search — finding code by meaning rather than exact keywords.

### Current Choice: Srclight

| Attribute | Value |
|-----------|-------|
| License | MIT |
| Stars | ~17 (very new) |
| Search | Hybrid (FTS5 trigram + semantic embeddings via RRF) |
| Parsing | tree-sitter (7 languages) |
| Embeddings | Ollama (qwen3-embedding default) |
| Database | SQLite |
| MCP Tools | 25 |
| Air-gap | Yes (Ollama models pre-downloaded) |

### Alternatives

| Tool | License | Hybrid Search | Embedding Support | C/C++ | Air-Gap | Maturity | MCP | Verdict |
|------|---------|---------------|-------------------|-------|---------|----------|-----|---------|
| **Srclight** | MIT | FTS5 + semantic (RRF) | Ollama (local) | Yes (tree-sitter) | Yes | Very new (17 stars) | Yes (native) | **Current choice** |
| **Zoekt** | Apache-2.0 | Trigram only (no semantic) | No | Yes | Yes | Mature (Google-backed) | No | Best keyword search; no semantic |
| **LanceDB** | Apache-2.0 | Vector + full-text | Any | Via embeddings | Yes | Growing | No (needs adapter) | Superior vector performance; no MCP |
| **ChromaDB** | Apache-2.0 | Semantic only | Yes | Via embeddings | Yes | Mature | No (needs adapter) | No FTS; pure vector |
| **Qdrant** | Apache-2.0 | Vector + filter | Yes | Via embeddings | Yes | Mature | No (needs adapter) | Best vector scaling; heavier |
| **Sourcegraph** | Apache-2.0 | Zoekt + SCIP | Extensions | Yes | Yes | Mature | Partial | Full platform; heavy |
| **Milvus** | Apache-2.0 | Hybrid | Yes | Via embeddings | Partial | Mature | No | Enterprise-grade; too heavy for local |

### Analysis

**Srclight is a reasonable choice with caveats**:

1. **Only MCP-native hybrid search tool** — combines FTS5 keyword + semantic embeddings + tree-sitter parsing in a single MCP server. No alternative provides all three as an MCP server.
2. **SQLite simplicity** — zero external dependencies. No Docker, no server process, no cluster management. Ideal for air-gapped.
3. **tree-sitter aware** — understands code structure (functions, classes, methods), not just text.

**However, significant maturity risk**:
- Only ~17 GitHub stars as of March 2026. Single contributor. Very new project.
- No independent benchmarks or production deployment reports found.
- If Srclight becomes unmaintained, the architecture needs a fallback.

**Strongest alternative combination**: Zoekt (keyword) + LanceDB (semantic) behind a custom MCP adapter.
- Zoekt provides battle-tested trigram search (used by Sourcegraph in production, Google-backed)
- LanceDB provides high-performance embedded vector search (Apache-2.0, Rust-based, zero-copy, very low resource usage)
- Both are fully offline and open source
- **Disadvantage**: Requires building a custom MCP server to combine them; no turnkey solution exists

**If maturity is paramount**: Sourcegraph (self-hosted) provides the most proven code search platform, but requires significant infrastructure (4-16GB RAM, Docker/Kubernetes) and is designed for teams, not single developers.

---

## Layer 5: Cross-Session Persistence (Memory)

The persistence layer stores decisions, insights, and context across sessions so the agent doesn't start from zero every time.

### Current Choice: Memora

| Attribute | Value |
|-----------|-------|
| License | MIT |
| Stars | ~321 |
| Backend | SQLite |
| Search | Hybrid (keyword + semantic) |
| MCP Server | Yes |
| Air-gap | Yes (cloud sync must be disabled) |
| Cloud Features | S3, R2, D1 sync (MUST be disabled) |

### Alternatives

| Tool | License | Offline | Hybrid Search | MCP Server | Cross-Session | Maturity | Verdict |
|------|---------|---------|---------------|------------|---------------|----------|---------|
| **Memora** | MIT | Yes (with config) | Yes | Yes | Yes | Moderate (321 stars) | **Current choice** |
| **Mem0** | Apache-2.0 | Yes (self-hosted OSS) | Yes (vector + graph) | Yes | Yes | High (26k+ stars) | Stronger community; more features |
| **Zep** | Apache-2.0 | Yes (self-hosted) | Yes | Yes | Yes | High | Temporal facts; heavier |
| **Serena memories** | MIT | Yes | No (text only) | Via Serena | Yes | Mature | Simple but no semantic search |
| **AGENTS.md files** | N/A | Yes | No | No | Manual only | N/A | Sufficient for small projects |
| **Custom SQLite** | N/A | Yes | Configurable | Custom | Yes | N/A | Maximum control; build effort |

### Analysis

**Memora is the correct choice for this architecture**, despite Mem0's larger community.

#### Memora vs Mem0: Detailed Trade-Off

**What Memora provides that Mem0 OSS does not:**

| Advantage | Detail |
|-----------|--------|
| **Pure SQLite, zero dependencies** | Single file, `pip install memora`, done. Mem0 OSS defaults to Qdrant (vector DB) + OpenAI (LLM + embeddings) + SQLite (history only). |
| **Local embeddings out of the box** | `pip install memora[embeddings]` adds sentence-transformers with no API key. Mem0 defaults to OpenAI embeddings (requires `OPENAI_API_KEY`). |
| **Native FTS5 keyword search** | SQLite FTS5 built-in alongside semantic search. Mem0 is semantic-first with no keyword fallback — exact matches can miss. |
| **Built-in Graph UI** | Browser-based knowledge graph visualization, timeline panel, chat panel with RAG, mermaid rendering. Mem0 OSS has no UI. |
| **Cloud sync as future option** | S3, R2, D1 available if air-gap requirements change. Disabled by default; risk is manageable via env var audit. |
| **Simpler deployment** | One process, one SQLite file. Mem0 self-hosted needs Docker Compose with 3 containers (Qdrant + Postgres + Mem0 API). |
| **Cross-references** | Auto-linked related memories based on similarity, typed edges, importance boosting, cluster detection. Limited in Mem0 OSS. |

**What Mem0 OSS provides that Memora does not:**

| Advantage | Detail |
|-----------|--------|
| **Community and maturity** | 26k+ stars, 160+ contributors, $24M funding vs Memora's 321 stars. |
| **Hierarchical memory scopes** | `user_id` / `session_id` / `agent_id` isolation. Better for multi-agent setups. |
| **Memory compression** | 80-90% token reduction via intelligent consolidation. Memora stores raw. |
| **Automatic entity extraction** | Extracts facts from conversations without explicit "save" calls. |
| **Contradiction handling** | Detects and resolves conflicting memories across sessions. |
| **24+ vector store backends** | Qdrant, Chroma, pgvector, Pinecone, Milvus, etc. Future flexibility. |
| **16+ LLM providers** | OpenAI, Anthropic, Ollama, Groq, local models for fact extraction. |

**Why Memora wins for air-gapped:**

1. **Mem0 OSS is not air-gap native.** Its default configuration requires an OpenAI API key for both the LLM (fact extraction) and embeddings. Reconfiguring to Ollama is possible but requires explicit setup, and there are known SQLite binding issues on some platforms (GitHub issue #4156).
2. **Mem0's extra features target SaaS applications.** Hierarchical scopes, entity extraction, and contradiction handling are designed for multi-user products serving thousands of users — not a single developer's coding agent.
3. **Deployment complexity.** Mem0 self-hosted recommends Docker Compose with Qdrant + PostgreSQL + Mem0 API server. Memora is one pip install and one SQLite file.
4. **The maturity gap does not affect functionality.** Memora does exactly what this stack needs: store memories, search them (keyword + semantic), recall them across sessions. The additional features Mem0 offers do not improve coding agent memory for this use case.

**Cloud sync risk is manageable.** The env vars that enable cloud sync (`MEMORA_STORAGE_URI`, `CLOUDFLARE_API_TOKEN`) are already documented in the [Security Assessment](./security-assessment.md) with explicit instructions to leave them unset.

**Simplest alternative**: Serena's built-in `.serena/memories/` (markdown files) combined with AGENTS.md provides basic cross-session persistence with zero additional infrastructure. Sufficient for projects where semantic memory search is not critical.

---

## Consolidated Comparison

### Current Architecture vs Best Alternatives

| Layer | Current Choice | Strongest Alternative | Switch Recommended? |
|-------|---------------|----------------------|-------------------|
| Base Agent | **OpenCode** | Goose (MIT, MCP, CLI) | **No** — OpenCode has 8x community, plugin system, native MCP |
| Orchestration | **OMO Slim** | OpenAgentsControl / Vanilla | **No** — OMO Slim has best token efficiency + parallel agents |
| Symbolic (LSP) | **Serena** | clangd-mcp-server | **No** — Serena's 30+ language support outweighs alternatives |
| Semantic (RAG) | **Srclight** | Zoekt + LanceDB | **Monitor** — Srclight is ideal but immature; have a fallback plan |
| Persistence | **Memora** | Mem0 (self-hosted OSS) | **No** — Memora is simpler, air-gap native, zero-dependency; Mem0's extras target SaaS, not coding agents |

### Risk Assessment

| Component | Risk Level | Risk Type | Mitigation |
|-----------|-----------|-----------|------------|
| OpenCode | Low | N/A | 124k stars, corporate backing, active development |
| OMO Slim | Medium | Fork sustainability | Monitor upstream oh-my-openagent; OMO Slim has 2.2k stars and 30 contributors |
| Serena | Medium | RAM usage at scale (30GB reported) | Use ccls for large repos; monitor GitHub issues |
| **Srclight** | **High** | **Single contributor, ~17 stars, 3 weeks old** | **Prepare Zoekt + LanceDB fallback; monitor adoption** |
| Memora | Low | Cloud sync misconfiguration (manageable) | Audit env vars per security-assessment.md; cloud sync disabled by default |

---

## Final Assessment

The current architecture represents the **best available combination** for the stated requirements (air-gapped, large codebase, open source, auditable), with one significant risk:

**Srclight** is the weakest link — technically sound (hybrid search with MCP is exactly right) but dangerously immature. A single-contributor project with 17 stars carries real abandonment risk. The architecture should include a documented fallback to Zoekt (keyword search) + LanceDB (vector search) behind a custom MCP adapter.

All other components are the clear best-in-class for their layer:

- **OpenCode**: No alternative matches its MCP-native + plugin system + terminal-first + MIT combination
- **OMO Slim**: Only token-optimized multi-agent orchestration plugin with air-gap configuration support
- **Serena**: Only mature MCP-native LSP wrapper supporting 30+ languages
- **Memora**: Pure SQLite, zero-dependency, air-gap native. Mem0 has a larger community (26k stars) but requires Qdrant + OpenAI by default — its extra features (hierarchical scopes, entity extraction, compression) target multi-user SaaS, not single-developer coding agents. Memora's simplicity is the right trade-off for this architecture.

**The architecture is not just good — it is currently the only fully integrated, MCP-native, air-gap-capable, open-source coding intelligence stack that exists.** Individual components have alternatives, but no other project assembles all five layers into a coordinated system.

## Related Documents

- [Architecture Overview](../architecture/overview.md)
- [Why Orchestration](./orchestration-vs-chat.md)
- [OMO Slim Air-Gap Analysis](./omo-slim-airgap-analysis.md)
- [Security Assessment](./security-assessment.md)
- [OpenCode Setup](../guides/opencode-setup.md)
