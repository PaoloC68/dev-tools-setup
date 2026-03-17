# Model-to-Agent Mapping: Optimal Configuration for Oh-My-OpenCode-Slim

## Available Models

### Specifications Summary

| Model | Architecture | Total Params | Active Params | Context | License | Key Strength |
|-------|-------------|-------------|---------------|---------|---------|-------------|
| **Qwen3.5-397B-A17B** | MoE (GDN + MoE) | 397B | 17B | 262k (1M w/ YaRN) | Apache 2.0 | Highest open-weight intelligence; multimodal |
| **Kimi-K2.5** | MoE | 1T | 32B | 256k | Modified MIT | Best tool calling; Agent Swarm; vision-to-code |
| **gpt-oss-120b** | MoE | 117B | 5.1B | 128k | Apache 2.0 | Best tool-use per active param; single H100 |
| **Nemotron-3-Super-120B-A12B-FP8** | Hybrid Mamba-2 + MoE | 120B | 12B | 1M | NVIDIA Open | Highest throughput; 1M context; agentic focus |
| **qwen3-coder-next** | MoE (GDN + MoE) | 80B | 3B | 262k | Open weights | Coding-specialized; ultra-efficient; no thinking mode |
| **Qwen3-30B-A3B-Instruct-2507** | MoE | 30.5B | 3.3B | 262k | Apache 2.0 | Smallest footprint; dual-mode (thinking/non-thinking) |
| **mistral-medium-2505** | Dense | Undisclosed | All (dense) | 128k | Proprietary (deployable) | Vision + text; function calling; 4-GPU deployment |

### Coding Benchmarks

| Model | SWE-bench Verified | LiveCodeBench v6 | HumanEval | MMLU-Pro | Tool-Use (TauBench) |
|-------|-------------------|-------------------|-----------|----------|-------------------|
| **Qwen3.5-397B-A17B** | 76.4% | 83.6% | — | 87.8% | 86.7% |
| **Kimi-K2.5** | 76.8% | 85.0% | — | 87.1% | 77.0% |
| **gpt-oss-120b** | 62.4% | 88.0%* | 88.3% | 81.0% | 61.0% |
| **Nemotron-3-Super-FP8** | 60.5% | 78.4% | — | 83.6% | 61.1% |
| **qwen3-coder-next** | 42.8% | — | — | — | — |
| **Qwen3-30B-A3B-2507** | — | — | — | ~61.5% | — |
| **mistral-medium-2505** | — | 30.3% | 92.1% | 77.2% | — |

*gpt-oss-120b LiveCodeBench score from NVIDIA's comparison; may use different version/conditions.

### Inference Characteristics

| Model | Min VRAM | Speed | Reasoning Modes | Function Calling |
|-------|---------|-------|----------------|-----------------|
| **Qwen3.5-397B-A17B** | ~214GB (Q4) | 8.6-19x faster than Qwen3-Max | Thinking + Non-Thinking | Yes |
| **Kimi-K2.5** | ~512GB (FP16) | Native INT4 = 2x speedup | Thinking + Instant | Yes (200-300 sequential calls stable) |
| **gpt-oss-120b** | 80GB (1x H100 MXFP4) | 400-500 tok/s (Cerebras) | Low / Medium / High reasoning | Yes (strong tool-use) |
| **Nemotron-3-Super-FP8** | 160GB (2x H100) | 2.2x faster than gpt-oss-120b | Configurable on/off | Yes |
| **qwen3-coder-next** | ~48GB (Q4) | Very fast (3B active) | Non-thinking only | Yes |
| **Qwen3-30B-A3B-2507** | ~24GB (Q4) | Fast (3.3B active) | Thinking + Non-Thinking | Yes |
| **mistral-medium-2505** | 4x GPU | Medium | No reasoning mode | Yes |

### Reasoning vs NoReasoning Variants

Several models offer reasoning-disabled variants. These skip chain-of-thought, producing faster and cheaper responses:

| Variant | Effect | Best For |
|---------|--------|---------|
| **Kimi-K2.5** (thinking enabled) | Full chain-of-thought reasoning | Complex planning, architecture review |
| **Kimi-K2.5-NoReasoning** | Direct response, no CoT | Simple retrieval, fast exploration |
| **Qwen3.5-397B-A17B** (thinking) | Full reasoning | Orchestration, planning |
| **Qwen3.5-397B-A17B-NoReasoning** | Direct response | Fast classification, routing decisions |
| **gpt-oss-120b-thinking-low** | Minimal reasoning | Quick tasks with some reasoning |
| **gpt-oss-120b-thinking-medium** | Moderate reasoning | Balanced speed/quality |

---

## Agent Requirements

Each Oh-My-OpenCode-Slim agent has different requirements:

| Agent | Role | Needs | Priority |
|-------|------|-------|----------|
| **Sisyphus** | Orchestrator | Best reasoning, delegation, tool calling, plan tracking | Quality > Speed |
| **Prometheus** | Planner | Deep reasoning, architecture understanding, task decomposition | Quality > Speed |
| **Oracle** | Advisor | Best reasoning, read-only analysis, debugging expertise | Quality > Speed |
| **Explorer** | Recon | Fast, cheap, good at search/grep patterns, parallel execution | Speed > Quality |
| **Librarian** | Knowledge | Fast, good at retrieval, documentation understanding | Speed > Quality |
| **Designer** | Visual | UI/UX understanding, visual reasoning, coding quality | Quality ≈ Speed |
| **Fixer** | Implementation | Strong coding, medium reasoning, fast turnaround | Speed ≈ Quality |

---

## Recommended Configuration

### Primary Recommendation

Optimized for quality + cost efficiency:

```jsonc
{
  "agents": {
    // Tier 1: Best reasoning (few calls, high impact)
    "sisyphus":  { "model": "Qwen3.5-397B-A17B" },
    "prometheus": { "model": "Kimi-K2.5" },
    "oracle":    { "model": "Qwen3.5-397B-A17B" },

    // Tier 2: Fast and cheap (many calls, parallel)
    "explorer":  { "model": "qwen3-coder-next" },
    "librarian": { "model": "Qwen3-30B-A3B-Instruct-2507" },

    // Tier 3: Balanced (moderate calls, needs coding quality)
    "designer":  { "model": "Kimi-K2.5-NoReasoning" },
    "fixer":     { "model": "gpt-oss-120b-thinking-low" }
  }
}
```

### Rationale

| Agent | Model | Why This Model |
|-------|-------|---------------|
| **Sisyphus** | Qwen3.5-397B-A17B | Highest MMLU-Pro (87.8%), best instruction following (IFBench 76.5%), strong agentic scores (TAU2-Bench 86.7%). The orchestrator needs the best overall intelligence to route correctly. |
| **Prometheus** | Kimi-K2.5 | Best math reasoning (AIME 96.1%), stable tool calling across 200-300 sequential calls, Agent Swarm capability for parallel task decomposition. Planner benefits from extended reasoning chains. |
| **Oracle** | Qwen3.5-397B-A17B | Same rationale as Sisyphus — best overall intelligence. Oracle is read-only consultation, so the highest-quality model prevents bad architectural advice. GPQA 88.4% (PhD-level science) ensures deep technical understanding. |
| **Explorer** | qwen3-coder-next | Only 3B active params = fastest inference, lowest cost. Coding-specialized with 262k context. No-thinking mode means zero overhead on search-and-return tasks. Designed explicitly for coding agents. |
| **Librarian** | Qwen3-30B-A3B-2507 | 3.3B active, extended to 262k context in July 2025 update. Dual-mode (can think when needed for complex doc analysis, skip thinking for simple retrieval). Apache 2.0 = fully auditable. |
| **Designer** | Kimi-K2.5-NoReasoning | Native vision-to-code capability — generates code directly from UI mockups, images, and video. NoReasoning for speed on design tasks where visual understanding matters more than deep logic. |
| **Fixer** | gpt-oss-120b-thinking-low | Strong HumanEval (88.3%), excellent tool-use, runs on single H100. Low-thinking mode provides enough reasoning for bug fixes without the overhead of full CoT. Apache 2.0. |

### Alternative Configuration: Throughput-Optimized

For hardware-constrained environments prioritizing speed:

```jsonc
{
  "agents": {
    "sisyphus":  { "model": "gpt-oss-120b-thinking-medium" },
    "prometheus": { "model": "gpt-oss-120b-thinking-medium" },
    "oracle":    { "model": "Nemotron-3-Super-120B-A12B-FP8" },
    "explorer":  { "model": "Qwen3-30B-A3B-Instruct-2507" },
    "librarian": { "model": "Qwen3-30B-A3B-Instruct-2507" },
    "designer":  { "model": "mistral-medium-2505" },
    "fixer":     { "model": "qwen3-coder-next" }
  }
}
```

| Agent | Model | Why |
|-------|-------|-----|
| **Sisyphus** | gpt-oss-120b-thinking-medium | Single H100, 400+ tok/s, strong tool-use. Medium reasoning balances quality/speed. |
| **Prometheus** | gpt-oss-120b-thinking-medium | Same model as Sisyphus — reduces model loading overhead. Shared GPU. |
| **Oracle** | Nemotron-3-Super-FP8 | 1M context window for deep analysis. 2.2x faster throughput than gpt-oss-120b. Excellent agentic scores. |
| **Explorer** | Qwen3-30B-A3B-2507 | Tiny footprint, fast, can run on 24GB GPU. Non-thinking mode for speed. |
| **Librarian** | Qwen3-30B-A3B-2507 | Same as Explorer — shared model instance. |
| **Designer** | mistral-medium-2505 | Multimodal (vision + text), 4-GPU deployment, strong on HumanEval. |
| **Fixer** | qwen3-coder-next | 3B active, coding-specialized, fastest turnaround for implementation tasks. |

### Alternative Configuration: Maximum Context

For very large codebases requiring deep context:

```jsonc
{
  "agents": {
    "sisyphus":  { "model": "Nemotron-3-Super-120B-A12B-FP8" },
    "prometheus": { "model": "Qwen3.5-397B-A17B" },
    "oracle":    { "model": "Nemotron-3-Super-120B-A12B-FP8" },
    "explorer":  { "model": "qwen3-coder-next" },
    "librarian": { "model": "Qwen3-30B-A3B-Instruct-2507" },
    "designer":  { "model": "Kimi-K2.5-NoReasoning" },
    "fixer":     { "model": "gpt-oss-120b-thinking-low" }
  }
}
```

Nemotron-3-Super for Sisyphus and Oracle provides **1M token context** — critical when the orchestrator needs to hold large tool results and the advisor needs to analyze extensive code sections simultaneously. Its 95.67% RULER@512k score means quality holds deep into long contexts (vs gpt-oss-120b at 46.70%).

---

## Model Suitability Matrix

How each model scores against each agent role (1-5, higher is better):

| Model | Sisyphus | Prometheus | Oracle | Explorer | Librarian | Designer | Fixer |
|-------|----------|-----------|--------|----------|-----------|----------|-------|
| Qwen3.5-397B-A17B | 5 | 4 | 5 | 2 | 2 | 4 | 3 |
| Kimi-K2.5 | 4 | 5 | 4 | 2 | 2 | 5 | 3 |
| gpt-oss-120b | 3 | 3 | 3 | 3 | 3 | 2 | 4 |
| Nemotron-3-Super-FP8 | 4 | 3 | 4 | 3 | 3 | 2 | 3 |
| qwen3-coder-next | 1 | 1 | 1 | 5 | 4 | 1 | 4 |
| Qwen3-30B-A3B-2507 | 1 | 1 | 1 | 4 | 5 | 1 | 3 |
| mistral-medium-2505 | 2 | 2 | 2 | 3 | 3 | 3 | 3 |

**Reading the matrix:**
- **5** = Optimal for this role
- **4** = Strong fit
- **3** = Adequate
- **2** = Suboptimal but functional
- **1** = Poor fit (wrong capability profile)

---

## Air-Gap Deployment Considerations

All models must be served locally via Ollama, vLLM, or similar. Key constraints:

| Model | Quantization Required | Local Serving Feasibility | Air-Gap Concern |
|-------|----------------------|--------------------------|-----------------|
| Qwen3.5-397B-A17B | Q4 (~214GB) or multi-GPU FP16 | Requires 256GB+ RAM or multi-GPU | Weights available on HuggingFace; no runtime downloads |
| Kimi-K2.5 | Q4 required (~512GB FP16) | Multi-node or high-end server | Large download; verify checksum before air-gap |
| gpt-oss-120b | MXFP4 on single H100 | Single H100 or 2x consumer GPU | Apache 2.0; weights on HuggingFace |
| Nemotron-3-Super-FP8 | FP8 native (no additional quant) | 2x H100-80GB minimum | NVIDIA Open License (check commercial terms) |
| qwen3-coder-next | Q4 (~48GB) | Single 24GB+ GPU or CPU offload | Open weights on HuggingFace |
| Qwen3-30B-A3B-2507 | Q4 (~24GB) | Single consumer GPU (RTX 4090) | Apache 2.0; smallest download |
| mistral-medium-2505 | FP16 on 4 GPUs | 4x GPU required | Proprietary model — check license for air-gap redistribution |

### Hardware Estimation for Primary Configuration

Running the recommended configuration requires serving 4 distinct models simultaneously:

| Model Instance | Serves Agents | Min VRAM |
|---------------|---------------|---------|
| Qwen3.5-397B-A17B | Sisyphus + Oracle | ~214GB (Q4) |
| Kimi-K2.5 | Prometheus + Designer (NoReasoning) | ~256GB (Q4) |
| gpt-oss-120b | Fixer | 80GB (1x H100) |
| qwen3-coder-next + Qwen3-30B-A3B | Explorer + Librarian | ~72GB combined (Q4) |

**Total estimated**: 600-800GB VRAM across a multi-GPU server, or distributed across multiple machines behind vLLM.

For smaller setups, the **throughput-optimized configuration** requires only 2 model instances (gpt-oss-120b + Qwen3-30B-A3B), fitting on ~104GB total VRAM.

## Related Documents

- [Architecture Overview](../architecture/overview.md)
- [OpenCode Setup](../guides/opencode-setup.md)
- [OMO Slim Air-Gap Analysis](./omo-slim-airgap-analysis.md)
- [Why Orchestration](./orchestration-vs-chat.md)
- [Comparative Analysis](./comparative-analysis.md)
