# AGENTS.md — dev-tools-setup

## Project Overview

Documentation repository for an **air-gapped AI-assisted coding architecture** — a 100% offline
RAG stack for C/C++ codebase intelligence using the Model Context Protocol (MCP).

Stack components documented here:
- **OpenCode** — Base terminal AI coding agent (anomalyco/opencode)
- **Oh-My-OpenCode-Slim** — Orchestration plugin that provides multi-agent hub-and-spoke coordination
- **Serena** — Symbolic/structural MCP server (LSP-based symbol navigation via clangd/ccls)
- **Srclight** — Semantic search MCP server (hybrid search: SQLite FTS5 + tree-sitter + Ollama embeddings)
- **Memora** — Persistence MCP server (cross-session memory, SQLite, cloud sync DISABLED for air-gap)

Architecture: hub-and-spoke. Oh-My-OpenCode-Slim orchestrates specialized agents (Sisyphus, Prometheus, Oracle, Explorer, Librarian, Designer, Fixer) that coordinate Serena, Srclight, and Memora via MCP.

## Repository Structure

```
docs/
  architecture/     # System architecture diagrams and overview
  guides/           # Setup and quickstart guides for each component
  research/         # Deep-dive technical references, security assessment, best practices
```

This repo contains **only Markdown documentation**. No source code, no build system, no tests.

## Build / Lint / Test Commands

No build or test pipeline exists. All content is Markdown.

### Validation (manual)

```bash
# Check for broken internal links (relative paths between docs)
grep -rn '](../' docs/ | while read line; do
  file=$(echo "$line" | cut -d: -f1)
  link=$(echo "$line" | grep -oP '\]\(\K[^)]+')
  dir=$(dirname "$file")
  [ ! -f "$dir/$link" ] && echo "BROKEN: $file -> $link"
done

# Optional: lint markdown with markdownlint if installed
npx markdownlint-cli2 "docs/**/*.md"
```

### Adding New Documentation

1. Determine category: `architecture/`, `guides/`, or `research/`
2. Create the `.md` file in the appropriate directory
3. Add cross-references to related docs using relative links
4. Verify all internal links resolve correctly

## Documentation Style Guide

### Language and Tone

- **Neutral, concise language** — no fluff, no filler
- **Direct, action-oriented** — "Copied" not "We copied", "Installed" not "We've installed"
- **No overly personal language** — avoid first-person in docs and comments
- **No flattery or filler phrases** — get to the point
- Guided by The Zen of Python: explicit > implicit, simple > complex, flat > nested

### Markdown Formatting

- H1 (`#`) for document title — one per file
- H2 (`##`) for major sections
- H3 (`###`) for subsections
- Use fenced code blocks with language identifiers (```bash, ```yaml, ```python, etc.)
- Use tables for structured comparisons (tool lists, feature matrices)
- Use ASCII diagrams for architecture visuals (box-drawing characters)
- Blank line before and after code blocks, tables, and headings
- No trailing whitespace

### Internal Links

- Always use **relative paths** between docs: `[Overview](../architecture/overview.md)`
- Link to related docs in a "Next Steps" or "Related Documents" section at file end
- Verify links resolve — broken links are unacceptable

### Document Structure Pattern

Every guide follows this structure (observe existing docs):

```
# Title
## Overview / What is X?
## Installation
## Configuration
## Usage / Quick Start
## Use Cases (with concrete examples)
## Troubleshooting
## Next Steps / Related Documents
```

### Code Examples

- Provide concrete, runnable examples — not pseudocode
- Include comments only when the code isn't self-explanatory
- Show CLI commands with `bash` fence, configs with `yaml`/`json` fence
- For multi-step workflows, number the steps

### Tables

- Use tables for tool references, feature comparisons, query routing
- Keep tables concise — avoid paragraph-length cells
- Align columns for readability in source

## Architecture Context for Agents

When editing docs, maintain consistency with these architectural facts:

| Component | Storage | Query Type | Example Query |
|-----------|---------|------------|---------------|
| Serena | `.serena/memories/` (markdown) | Symbolic | "Find function login" |
| Srclight | `.srclight/index.db` (SQLite + FTS5 + embeddings) | Hybrid (keyword + semantic) | "Where is auth handled?" |
| Memora | local SQLite (cloud sync disabled) | Historical + semantic | "What did we decide?" |

Key technical details to preserve:
- Srclight embeddings: Ollama with `qwen3-embedding` (default) or `nomic-embed-text` (lighter)
- Srclight search: Hybrid — FTS5 trigram + semantic via Reciprocal Rank Fusion (RRF)
- Srclight parsing: tree-sitter (7 languages: C, C++, Python, TypeScript, JavaScript, Rust, Go)
- C/C++ LSP: clangd (default, recommended) or ccls (alternative)
- `compile_commands.json` is a hard requirement for C/C++ projects
- All components operate 100% offline — no runtime network access
- Serena installation: use `uvx` (recommended) or `pip`; avoid `npx -y` in air-gapped
- Memora cloud sync (S3, R2, D1) must remain disabled for air-gapped deployments
- MCP latency: ~50-200ms per call
- Token efficiency: ~70% savings over text-based RAG

## Security Awareness

This is an air-gapped stack. When editing docs:
- Never recommend cloud services or external API calls
- Never assume network access in setup instructions
- Always include air-gap warnings when documenting features that could phone home
- Reference `docs/research/security-assessment.md` for the full security analysis
- Serena has known vulnerabilities (shell execution, 0.0.0.0 binding) — always document mitigations
- `npx -y` always downloads — never recommend it for air-gapped environments

## Naming Conventions

- File names: lowercase, hyphen-separated (e.g., `serena-quickstart.md`)
- Section IDs: derived from headings automatically — keep headings stable for link anchors
- Component names: capitalize as proper nouns (Serena, Srclight, Memora, OpenCode)
- Config paths: use dotfiles (`.serena/`, `.srclight/`, `.memora/`, `.opencode/`)

## Error Handling in Documentation

- When documenting error scenarios, include the exact error message
- Provide concrete fix steps, not vague suggestions
- Use a `## Troubleshooting` section with `### "Error message"` subsections
- Pattern: error message heading -> cause -> fix command/steps

## Commit Messages

- No AI metadata (no robot emoji, no `Co-Authored-By: Claude`)
- Concise, imperative mood: "Add Srclight GPU setup guide" not "Added guide for..."
- Focus on what changed, not how it was generated

## Common Pitfalls

- Don't reference cloud services or external APIs — this is an air-gapped stack
- Don't assume network access in any setup instructions
- Don't use `all-MiniLM-L6-v2` as the Srclight embedding model — the correct default is `qwen3-embedding` via Ollama
- Don't recommend `npx -y` for air-gapped setups — it always downloads from npm
- Don't omit air-gap warnings when documenting cloud-capable features (e.g., Memora sync)
- Don't add unnecessary abstraction layers to documentation structure
- Don't duplicate content across guides — cross-reference instead
- Keep architecture diagrams consistent across all docs (same box-drawing style)
- When adding new components, update `docs/architecture/overview.md`