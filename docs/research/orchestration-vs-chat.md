# Why Orchestration: Multi-Agent vs Single-Agent for Codebase Intelligence

## The Problem

An AI coding assistant needs two things simultaneously:

1. **Project awareness** — understanding of the overall architecture, conventions, and dependencies
2. **Focused context** — precise, relevant details about the specific code being modified

A single chat agent with MCP tools connected can provide either, but not both effectively. As the codebase grows, the tension between these two needs becomes the defining constraint.

This document examines why an orchestration layer (such as Oh-My-OpenCode-Slim) is not merely convenient but architecturally necessary for reliable AI-assisted development at scale.

## 1. The Context Window Is Not Unlimited

### The Numbers

Modern LLMs advertise large context windows (200k-2M tokens), but effective capacity is much smaller:

| Model | Claimed Window | Effective Limit (before quality degrades) |
|-------|---------------|------------------------------------------|
| Claude Opus 4 | 200k tokens | 60k-120k tokens |
| GPT-5.2 | 1M tokens | ~200k tokens |
| Gemini 2.5 Pro | 2M tokens | ~200k tokens |

Source: NoLiMa benchmark — 11 of 12 tested LLMs drop below 50% of their short-context performance at 32k tokens.

### What Fills the Context

In a single-agent setup with MCP tools, the context fills rapidly before any useful work begins:

| Consumer | Tokens | Notes |
|----------|--------|-------|
| MCP tool descriptions (50+ tools) | ~12k | Loaded on every message |
| Built-in tool descriptions | ~12k | Always present |
| System prompt + AGENTS.md | ~3-5k | Per session |
| Conversation history (10 turns) | ~20-40k | Grows linearly |
| File contents (3-5 files loaded) | ~15-50k | Depends on file size |
| **Total before doing work** | **~60-120k** | **Already at effective limit** |

A single agent navigating a large codebase (>100k files) will saturate its context window within 10-15 tool calls — long before it has enough information to make a correct change.

### Context Rot

Research demonstrates that LLM quality degrades as context fills, following predictable patterns:

- **<50% full**: U-shaped degradation — the model loses information in the middle of the context (Liu et al., 2023, "Lost in the Middle")
- **>50% full**: Recency bias — the model favors recent tokens, losing early context (Veseli et al., 2025)
- **>80% full**: Severe degradation — reasoning quality drops sharply regardless of information position

Source: Chroma Research, "Context Rot: How Increasing Input Tokens Impacts LLM Performance" (2025)

**Consequence for coding**: An agent that loaded the architecture overview, then 5 files, then ran some searches, will have lost its understanding of the architecture by the time it writes code. The early context (project structure, conventions) gets pushed out by recent context (specific file contents).

## 2. Single Agent: The Overloaded Generalist

A single chat agent connected to MCP tools attempts to be planner, implementer, reviewer, researcher, and debugger simultaneously. This creates specific failure modes:

### 2.1 Tool Selection Degradation

With 50+ MCP tools available, the agent must choose the right tool for each step. Research shows error rates in tool selection rise 15-25% beyond 20-30 available tools, due to description noise and choice overload.

In practice: an agent with Serena (13 tools) + Srclight (25 tools) + Memora (4 tools) + built-in tools (12) = **54 tools**. The tool descriptions alone consume ~24k tokens, and the agent frequently selects suboptimal tools or hallucinates tool calls that don't exist.

### 2.2 Task Switching Cost

A single agent switching between planning, coding, reviewing, and debugging must retain all phases in context simultaneously:

```
Turn 1-5:   Planning (loads architecture, makes decisions)
Turn 6-15:  Implementing (loads files, writes code)
Turn 16-20: Reviewing (re-reads what was written)
Turn 21-25: Debugging (loads test output, traces errors)
```

By turn 20, the planning context from turns 1-5 is buried deep in the conversation history — precisely where context rot causes the most loss. The agent "forgets" its own plan.

### 2.3 Compaction Destroys Project Awareness

When the context fills, the agent compacts (summarizes) its conversation history. This process systematically loses:

- Architectural decisions made early in the session
- Subtle constraints discovered during exploration
- Cross-file dependency chains
- The "why" behind design choices (only the "what" survives summarization)

After compaction, the agent often contradicts its own earlier decisions because it no longer remembers making them.

## 3. Multi-Agent Orchestration: Division of Labor

An orchestration layer addresses these problems by distributing work across specialized agents, each with its own context window and tool set.

### 3.1 The Hub-and-Spoke Model

```
                     User Request
                          │
                          ▼
              ┌───────────────────────┐
              │  Orchestrator (Hub)   │
              │  - Routes tasks       │
              │  - Maintains plan     │
              │  - Tracks progress    │
              └───────────┬───────────┘
                          │
          ┌───────┬───────┼───────┬───────┐
          ▼       ▼       ▼       ▼       ▼
       Planner  Explorer  Coder  Reviewer Researcher
       (Spoke)  (Spoke)  (Spoke) (Spoke)  (Spoke)
```

Each spoke agent:
- Receives only the context it needs (not the full conversation)
- Has access to only the tools relevant to its role (not all 54)
- Returns a concise result to the hub (not its full reasoning trace)
- Operates in its own context window (no interference from other tasks)

### 3.2 Context Isolation Solves Context Rot

The core insight: **each agent gets a fresh, focused context window.**

| Agent | Context Contains | Approximate Tokens | Tools Available |
|-------|-----------------|-------------------|----------------|
| Orchestrator | Plan + agent results + user request | ~15k | Task delegation only |
| Explorer | Search query + search results | ~8k | grep, glob, read |
| Planner | Architecture overview + requirements | ~20k | read, think |
| Coder | Specific files + symbol context | ~30k | edit, write, LSP |
| Reviewer | Changed files + test results | ~20k | read, LSP diagnostics |
| Researcher | Documentation query + retrieved docs | ~10k | websearch, context7 |

Compare to the single agent: **~120k tokens** to hold all of this simultaneously, with severe context rot. With orchestration: each agent uses **8-30k tokens**, well within the high-quality zone.

### 3.3 Parallel Execution

A single agent is sequential — it can only do one thing at a time. An orchestrated system runs agents in parallel:

```
Time  Single Agent              Multi-Agent System
─────┬──────────────────────── ┬────────────────────────────
 0s  │ Search for auth code    │ Explorer 1: Search auth code
     │                         │ Explorer 2: Search test patterns
     │                         │ Librarian: Find JWT docs
 5s  │ Read search results     │ (all three return results)
10s  │ Search for test patterns│ Planner: Create implementation plan
15s  │ Read search results     │
20s  │ Look up JWT docs        │ Coder: Implement (has plan + context)
25s  │ Read JWT docs           │
30s  │ Plan implementation     │ Reviewer: Check implementation
35s  │ Start coding            │ (Done)
...  │ ...                     │
60s  │ Finish coding           │
65s  │ Review own code         │
70s  │ (Done)                  │
```

Wall-clock time roughly halves for complex tasks. Token cost may increase but work quality improves because each agent operates in a clean context.

## 4. Model Routing: Right Model for the Right Job

A single agent uses one model for everything. An orchestrated system routes different models to different roles:

| Role | Model Class | Why | Cost per 1M tokens |
|------|------------|-----|---------------------|
| Orchestrator | Opus (expensive, best reasoning) | Needs to understand the full picture | $15-75 |
| Planner | Opus (expensive, best reasoning) | Architecture decisions require deep thinking | $15-75 |
| Explorer | Haiku (cheap, fast) | Just searching and returning results | $0.25-1 |
| Librarian | Haiku (cheap, fast) | Retrieval doesn't need deep reasoning | $0.25-1 |
| Coder | Sonnet (mid-tier) | Good balance of quality and cost | $3-15 |
| Reviewer | Sonnet (mid-tier) | Needs quality but not Opus-level | $3-15 |

**Cost impact**: If 60% of agent invocations are exploration/research (Haiku) and only 10% are planning (Opus), the weighted cost drops dramatically compared to running Opus for everything.

Real-world data (Vercel AI Gateway case study): model routing achieves ~70% cost reduction with equivalent output quality on coding tasks.

## 5. RAG Layers: How Orchestration Enables Project Awareness

The orchestration layer doesn't just delegate tasks — it manages how context is assembled for each agent. This is where the MCP servers (Serena, Srclight, Memora) become critical:

### 5.1 Three Layers of Context

| Layer | MCP Server | What It Provides | Token Cost |
|-------|-----------|-----------------|------------|
| **Symbolic** | Serena (LSP) | Function signatures, callers, callees, type info | ~500-2k per symbol |
| **Semantic** | Srclight (embeddings) | Relevant code snippets by meaning | ~1-5k per query |
| **Historical** | Memora (memory) | Past decisions, architecture insights, session context | ~500-2k per recall |

### 5.2 Symbol-Level vs File-Level Loading

A single agent typically loads entire files:

```
read_file("auth/handler.py")  → 8,000 tokens (entire file)
```

An orchestrated agent with Serena loads only the relevant symbol:

```
find_symbol("validate_token")  → 400 tokens (function + signature)
```

**Token efficiency: 10-25x reduction** for the same information. The Qwen3-Coder study (JetBrains, 2025) measured this precisely: symbol-level retrieval saves 52% of tokens while improving solve rates by 2.6%.

### 5.3 The Orchestrator's Context Assembly Role

The orchestrator decides what context each agent needs:

```
User: "Add JWT refresh token support"

Orchestrator's delegation:
├── Explorer: "Find existing auth middleware patterns"
│   → Returns: 3 file paths + pattern description (~500 tokens)
├── Librarian: "Find JWT refresh token best practices"
│   → Returns: Security recommendations (~800 tokens)
├── Planner receives: Explorer results + Librarian results + architecture memory
│   → Returns: Implementation plan (~1,500 tokens)
└── Coder receives: Plan + relevant symbols from Serena + specific files
    → Has focused, relevant context (~15k tokens total)
```

Without orchestration, a single agent would load all of this into one context window (~50k+ tokens), with the earliest exploration results lost to context rot by the time it starts coding.

## 6. Cross-Session Persistence: Why Memory Matters

A single chat session has no memory of previous sessions. Every new conversation starts from zero — the agent re-explores the codebase, re-discovers conventions, re-makes architectural decisions.

With a persistence layer (Memora):

| Scenario | Without Memory | With Memory |
|----------|---------------|-------------|
| Opening a project after 2 days | Agent explores for 5-10 minutes, may reach different conclusions | Agent loads prior decisions in ~2 seconds |
| Recalling why a design decision was made | Lost — must re-derive from code | Retrieved from session history |
| Applying project conventions | Discovered per-session via exploration | Loaded once, persisted across sessions |
| Multi-day refactoring | Each session starts fresh, risking inconsistency | Continuous context across sessions |

Memory transforms AI coding from stateless per-session interactions into a persistent partnership that accumulates understanding over time.

## 7. When Single-Agent Is Sufficient

Orchestration is not always necessary. A single agent with MCP tools works well for:

| Scenario | Why Single Agent Works |
|----------|----------------------|
| Small codebases (<10k lines) | Entire project fits in context without rot |
| Single-file edits | No cross-file context needed |
| Quick questions ("what does this function do?") | Minimal tool calls required |
| Well-scoped tasks with clear file targets | No exploration or planning needed |
| Interactive debugging with user guidance | Human provides the context management |

The break-even point is roughly: **when the task requires understanding more code than fits in one context window while maintaining quality**, orchestration becomes necessary.

## 8. When Orchestration Is Necessary

| Signal | Why Orchestration Helps |
|--------|------------------------|
| Codebase >50k lines | Too large for single-context exploration |
| Cross-cutting changes (>5 files) | Requires parallel context across files |
| Architectural modifications | Needs global awareness + local precision simultaneously |
| Multi-step workflows (plan → implement → test → review) | Each phase pollutes context for the next |
| Team environments | Shared model serving, consistent conventions |
| Long-running sessions (>30 minutes) | Context rot degrades single-agent quality |
| Cost-sensitive deployments | Model routing reduces token spend |

## 9. Quantitative Summary

| Metric | Single Agent + MCP | Orchestrated Multi-Agent | Source |
|--------|-------------------|-------------------------|--------|
| Effective context per task | 60-120k tokens (shared) | 8-30k per agent (isolated) | NoLiMa, Chroma Research |
| Context quality at 80% fill | <50% of baseline | N/A (agents stay <50%) | Lost in the Middle (Liu et al.) |
| Tool selection accuracy (50+ tools) | 75-85% | >90% (5-15 tools per agent) | Tool-augmented LLM research |
| Token cost per complex task | Baseline (1x) | 0.3-0.7x (model routing) | Vercel AI Gateway case study |
| Wall-clock time (complex tasks) | Baseline (1x) | 0.5-0.7x (parallelism) | Industry benchmarks |
| Cross-session continuity | None | Full (via persistence layer) | By design |
| SWE-bench improvement (scaffolding) | Baseline | +23-30% on complex tasks | Deloitte, Anthropic |

## 10. Conclusion

The fundamental issue is not whether an LLM is "smart enough" to code — modern models score 80%+ on SWE-bench when properly scaffolded. The issue is **context management**: how to deliver the right information to the model at the right time, without overwhelming its effective capacity.

A single chat agent with MCP tools provides raw access to code intelligence but places the entire burden of context management on one context window. As codebase size and task complexity grow, this approach hits hard limits:

- Context rot degrades quality predictably and unavoidably
- Tool proliferation reduces selection accuracy
- Task switching loses earlier reasoning
- Compaction destroys architectural awareness
- No cross-session memory means repeated exploration

An orchestration layer solves these by treating context as a managed resource:

- **Isolation**: each agent gets a fresh, focused context
- **Routing**: the right model and tools for each subtask
- **Parallelism**: multiple agents work simultaneously
- **Persistence**: decisions survive across sessions
- **Assembly**: the orchestrator composes context from multiple sources (LSP, embeddings, memory) tailored to each agent's needs

The result is not just faster or cheaper — it is qualitatively different. The system maintains project awareness (via the orchestrator and memory layer) while delivering focused context (via per-agent isolation and RAG), achieving what a single agent cannot: reliable AI-assisted development at scale.

## Related Documents

- [Architecture Overview](../architecture/overview.md)
- [OMO Slim Air-Gap Analysis](./omo-slim-airgap-analysis.md)
- [Security Assessment](./security-assessment.md)
- [OpenCode Setup](../guides/opencode-setup.md)

## References

- Liu et al. (2023). "Lost in the Middle: How Language Models Use Long Contexts." TACL, 12:157-173.
- Veseli et al. (2025). Context rot patterns beyond U-shape. Chroma Research.
- Chroma Research (2025). "Context Rot: How Increasing Input Tokens Impacts LLM Performance."
- JetBrains Research (2025). "Efficient Context Management for Coding Agents." Observation masking analysis.
- SambaNova (2025). "Are LLMs Truly Solving Software Problems?" SWE-bench long-context analysis.
- Anthropic (2025). "Building Effective Agents" and "Multi-Agent Research System."
- Google Developers Blog (2025). "Architecting Efficient Context-Aware Multi-Agent Framework."
- Deloitte (2025). "Autonomous Generative AI Agents Still Under Development."
- Vercel (2025). "How I Use OpenCode with Vercel AI Gateway to Build Features Fast."
- MorphLLM (2026). "We Tested 15 AI Coding Agents — Only 3 Changed How We Work."
