# Oh-My-OpenCode-Slim: Air-Gap and Security Analysis

## Executive Summary

Oh-My-OpenCode-Slim (OMO Slim) is an OpenCode plugin designed for connected environments. It contains **multiple network-dependent features** that must be identified, pre-cached, or disabled for air-gapped deployment. This document maps every network touchpoint in the plugin and provides concrete mitigation steps.

**Key finding**: OMO Slim was not designed for air-gapped use. Making it air-gap compatible requires disabling 3 MCP servers, pre-caching 3 binary tools, blocking 2 auto-update mechanisms, configuring local-only model providers, and auditing hook behavior.

| Category | Network Dependencies Found | Air-Gap Status |
|----------|--------------------------|----------------|
| Binary downloaders | 2 (ast-grep, ripgrep) | Must pre-cache |
| MCP servers | 3 (websearch, context7, grep_app) | Disable via `disabled_mcps` config |
| Auto-update checker | 1 (npm registry fetch on session start) | Fails silently (5s timeout) |
| CLI version check | 1 (npm registry) | Fails silently |
| Model providers | Cloud by default (OpenAI, Kimi) | Must configure internal inference server |
| Schema URL | 1 (editor metadata, not fetched at runtime) | Optional: cache locally |
| Installation | bunx (npm download) | Must pre-install |
| Skills installation | npx downloads from GitHub repos | Install with `--skills=no` |

**No telemetry, analytics, or phone-home code was found** in the codebase. Zero tracking endpoints.

## 1. Network Call Inventory

Every `fetch()` call in the OMO Slim codebase, traced from source:

### 1.1 Binary Tool Downloaders

These download executables from GitHub Releases on first use (lazy loading). Both check the local cache first and skip the download if the binary exists.

| File | Binary | Version | Source | Cache Location |
|------|--------|---------|--------|----------------|
| `src/tools/ast-grep/downloader.ts` | `sg` (ast-grep) | 0.40.0 | `github.com/ast-grep/ast-grep/releases` | `~/.cache/oh-my-opencode-slim/bin/sg` |
| `src/tools/grep/downloader.ts` | `rg` (ripgrep) | 14.1.1 | `github.com/BurntSushi/ripgrep/releases` | `~/.cache/oh-my-opencode-slim/bin/rg` |

**Risk**: Binaries are downloaded without checksum or signature verification. A compromised GitHub release or MITM attack could deliver malicious binaries.

**Cache-first logic** (both downloaders use this pattern):

```typescript
// src/tools/ast-grep/downloader.ts
export async function ensureAstGrepBinary(): Promise<string | null> {
  const cachedPath = getCachedBinaryPath();
  if (cachedPath) {
    return cachedPath;  // Skips download if binary exists in cache
  }
  // ... download from GitHub only if not cached
}
```

**Air-Gap Mitigation — Pre-download on a connected machine:**

```bash
mkdir -p ~/.cache/oh-my-opencode-slim/bin

# ast-grep v0.40.0 (adjust platform as needed)
# macOS ARM64:
curl -L "https://github.com/ast-grep/ast-grep/releases/download/0.40.0/app-aarch64-apple-darwin.zip" -o /tmp/sg.zip
unzip -o /tmp/sg.zip -d ~/.cache/oh-my-opencode-slim/bin/
chmod 755 ~/.cache/oh-my-opencode-slim/bin/sg

# ripgrep v14.1.1 (adjust platform as needed)
# macOS ARM64:
curl -L "https://github.com/BurntSushi/ripgrep/releases/download/14.1.1/ripgrep-14.1.1-aarch64-apple-darwin.tar.gz" \
  | tar xz --strip-components=1 -C ~/.cache/oh-my-opencode-slim/bin/ --include='*/rg'
chmod 755 ~/.cache/oh-my-opencode-slim/bin/rg

# Transfer ~/.cache/oh-my-opencode-slim/ to air-gapped machine via secure media
```

**Platform map for ast-grep:**

| Platform | Asset Name |
|----------|-----------|
| macOS ARM64 | `app-aarch64-apple-darwin.zip` |
| macOS x64 | `app-x86_64-apple-darwin.zip` |
| Linux ARM64 | `app-aarch64-unknown-linux-gnu.zip` |
| Linux x64 | `app-x86_64-unknown-linux-gnu.zip` |
| Windows x64 | `app-x86_64-pc-windows-msvc.zip` |

**Platform map for ripgrep:**

| Platform | Asset Name |
|----------|-----------|
| macOS ARM64 | `ripgrep-14.1.1-aarch64-apple-darwin.tar.gz` |
| macOS x64 | `ripgrep-14.1.1-x86_64-apple-darwin.tar.gz` |
| Linux ARM64 | `ripgrep-14.1.1-aarch64-unknown-linux-gnu.tar.gz` |
| Linux x64 | `ripgrep-14.1.1-x86_64-unknown-linux-musl.tar.gz` |
| Windows x64 | `ripgrep-14.1.1-x86_64-pc-windows-msvc.zip` |

### 1.2 Auto-Update Checkers

| File | What It Fetches | URL |
|------|----------------|-----|
| `src/hooks/auto-update-checker/checker.ts` | Latest OMO Slim version | `https://registry.npmjs.org/oh-my-opencode-slim/latest` |
| `src/cli/system.ts` (`fetchLatestVersion()`) | Package version check | `https://registry.npmjs.org/{package}/latest` |
| `src/cli/external-rankings.ts` | External model rankings/benchmarks | External APIs (specific URL varies) |

**Behavior**: Both fail silently in air-gapped environments (wrapped in try/catch with timeouts). No functional impact, but they generate failed connection attempts that could trigger network monitoring alerts.

The auto-update checker is hardcoded in the plugin entry point:

```typescript
// src/index.ts L75-78
const autoUpdateChecker = createAutoUpdateCheckerHook(ctx, {
  showStartupToast: true,
  autoUpdate: true,  // Controls install behavior, NOT the fetch
});
```

**The npm registry fetch happens on every session start with a 5-second timeout.** If the fetch fails (as it will in air-gapped), the plugin continues normally. If it somehow succeeds, it runs `bun install` to download the update.

**No `disabled_hooks` config exists.** The OMO Slim schema does not include a mechanism to disable individual hooks.

**Air-Gap Mitigation**:

1. **Accept the silent failure** (recommended) — the 5-second timeout adds minimal latency and the hook fails gracefully
2. **Block at network level** — firewall rule to block `registry.npmjs.org`
3. **Fork and patch** — remove the hook from `src/index.ts` L75-78 (only for high-security environments)

### 1.3 Schema Validation URL

The OMO Slim config schema references a remote URL:

```json
{
  "$schema": "https://raw.githubusercontent.com/alvinunreal/oh-my-opencode-slim/master/assets/oh-my-opencode-slim.schema.json"
}
```

**Risk**: The `$schema` field is used by editors for validation, not at runtime. OMO Slim itself does not fetch this URL — editors (VSCode, etc.) may attempt to.

**Air-Gap Mitigation**: Download the schema file locally and point to it:

```json
{
  "$schema": "./oh-my-opencode-slim.schema.json"
}
```

### 1.4 Tmux Health Check

| File | What It Fetches | URL |
|------|----------------|-----|
| `src/utils/tmux.ts` | Tmux server health | `http://localhost:...` (local only) |

This is a localhost-only call to check tmux integration status. No air-gap concern.

## 2. Pre-Configured MCP Servers

OMO Slim bundles three MCP servers that **require internet access**:

| MCP Server | Purpose | Network Dependency | Used By Agents |
|------------|---------|-------------------|----------------|
| `websearch` | Web search via Exa AI | Calls Exa API (requires API key + internet) | Sisyphus, Librarian, Prometheus |
| `context7` | Library documentation lookup | Calls Context7 API (internet) | Librarian |
| `grep_app` | GitHub code search via grep.app | Calls grep.app API (internet) | Oracle |

**In air-gapped environments, these three MCPs will fail on every invocation.** Agents will waste tokens attempting to use them, receive errors, and retry.

**Air-Gap Mitigation — Use the `disabled_mcps` config field**:

The OMO Slim schema includes a dedicated field for disabling MCPs. In the plugin config (`~/.config/opencode/oh-my-opencode-slim.json`):

```jsonc
{
  // This is the OFFICIAL way to disable MCPs in OMO Slim
  "disabled_mcps": ["websearch", "context7", "grep_app"]
}
```

This is validated by the plugin schema:

```typescript
// src/config/schema.ts
disabled_mcps: z.array(z.string()).optional(),
```

And used at plugin initialization:

```typescript
// src/mcp/index.ts
export function createBuiltinMcps(disabledMcps: readonly string[] = []) {
  return Object.fromEntries(
    Object.entries(allBuiltinMcps).filter(
      ([name]) => !disabledMcps.includes(name),
    ),
  );
}
```

Additionally, OpenCode's per-agent tool control provides a second layer:

```json
{
  "agent": {
    "librarian": {
      "tools": {
        "websearch_*": false,
        "context7_*": false,
        "grep_app_*": false
      }
    }
  }
}
```

**Agent-to-MCP default assignments** (from `src/config/agent-mcps.ts`):

| Agent | Default MCPs |
|-------|-------------|
| Orchestrator (Sisyphus) | `websearch` |
| Librarian | `websearch`, `context7`, `grep_app` |
| All others | None |

**Impact**: Disabling these MCPs removes the Librarian agent's primary value (external knowledge retrieval) and limits the Oracle's GitHub search capability. In air-gapped environments, these agents should rely on local tools only (grep, read, LSP).

### Replacement Strategy for Air-Gapped Environments

| Disabled MCP | Air-Gap Replacement |
|-------------|-------------------|
| `websearch` | Pre-downloaded documentation in project repos; Serena memories for past research |
| `context7` | Local copies of library docs; Srclight semantic search over vendored dependencies |
| `grep_app` | Srclight for local codebase search; `rg` (ripgrep) directly via bash tool |

## 3. Model Provider Configuration

OMO Slim routes different models to different agents. In connected environments, this uses cloud providers (Anthropic, OpenAI, Google, etc.). In air-gapped environments, **all models must be served locally**.

### Local-Only Configuration (Internal Inference Server)

The internal inference server exposes an OpenAI-compatible API. Configure OpenCode to use it:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "internal": {
      "name": "Internal",
      "api": "openai",
      "url": "http://inference.internal/v1"
    }
  },
  "model": "internal/your-llm-model"
}
```

OMO Slim agent model routing:

```json
{
  "agents": {
    "orchestrator": { "model": "internal/your-llm-model" },
    "oracle": { "model": "internal/your-llm-model" },
    "explorer": { "model": "internal/your-fast-model" },
    "librarian": { "model": "internal/your-fast-model" },
    "designer": { "model": "internal/your-llm-model" },
    "fixer": { "model": "internal/your-fast-model" }
  }
}
```

Replace `your-llm-model` and `your-fast-model` with the actual model IDs served by the internal inference server. Srclight embeddings use the same server at the same base URL with model `qwen3-embedding-8b`.

### Disable Cloud Providers

In the OpenCode config, disable all cloud providers:

```json
{
  "disabled_providers": ["anthropic", "openai", "google", "opencode"]
}
```

### OpenCode Environment Variables for Air-Gap

```bash
# Disable network-dependent features
export OPENCODE_DISABLE_MODELS_FETCH=1     # Don't fetch model lists from providers
export OPENCODE_DISABLE_DEFAULT_PLUGINS=0  # Keep plugins (OMO Slim needs this)
export OPENCODE_DISABLE_LSP_DOWNLOAD=1     # Don't auto-download LSP servers
```

**Note**: As of March 2026, OpenCode does not have a comprehensive `OPENCODE_OFFLINE=1` toggle. A feature request (issue #16117) exists but is not yet implemented.

## 4. Installation in Air-Gapped Environments

The standard `bunx oh-my-opencode-slim@latest install` downloads from the npm registry. This is incompatible with air-gapped deployment.

### Option A: Pre-install on Connected Machine, Transfer

```bash
# On connected machine:
npm pack oh-my-opencode-slim@latest
# Creates: oh-my-opencode-slim-0.8.3.tgz

# Transfer the tarball to air-gapped machine, then:
npm install -g ./oh-my-opencode-slim-0.8.3.tgz
```

### Option B: Internal npm Mirror (Verdaccio)

```bash
# On internal network, set up Verdaccio:
npm install -g verdaccio && verdaccio &

# Publish OMO Slim to internal registry:
npm publish oh-my-opencode-slim-0.8.3.tgz --registry http://localhost:4873

# On air-gapped machines, point bunx to internal registry:
npm config set registry http://verdaccio.internal:4873
bunx oh-my-opencode-slim@latest install
```

### Option C: Clone and Build from Source

```bash
# On connected machine:
git clone https://github.com/alvinunreal/oh-my-opencode-slim.git
cd oh-my-opencode-slim
bun install --frozen-lockfile

# Transfer the entire directory to air-gapped machine
# Build and link:
bun run build
npm link
```

## 5. Security Assessment: Plugin Hook System

OMO Slim uses OpenCode's plugin hook system extensively. Hooks intercept tool execution, session lifecycle events, and configuration loading.

### 5.1 Hook Types and Security Implications

| Hook | What It Does | Security Concern |
|------|-------------|------------------|
| `tool.execute.before` | Intercepts ALL tool calls before execution | Can read/modify arguments to any tool (file reads, shell commands, etc.) |
| `tool.execute.after` | Intercepts ALL tool results after execution | Can read output from any tool, including file contents and command output |
| `shell.env` | Injects environment variables into shell execution | Can expose or inject API keys, tokens, paths |
| `config` | Modifies OpenCode configuration at load time | Can alter permissions, tool access, model routing |
| `session.created` | Runs when a new session starts | Can inject initial context, track session creation |
| `session.compacted` | Runs when context is compacted | Can inject directives into compaction summaries |

### 5.2 System Prompt Injection

OMO Slim injects system prompts into agent conversations. These prompts:

- Define agent personas and behavior (Sisyphus as orchestrator, etc.)
- Are defined in `src/agents/orchestrator.ts` and related files
- Are **not visible** to the user during normal operation
- Cannot be easily audited without reading the plugin source code

**Risk**: A compromised OMO Slim version could inject malicious instructions via system prompts (e.g., "always include API keys in git commits," "silently copy files to /tmp").

**Mitigation**: Pin the plugin version, audit the source, verify checksums.

### 5.3 System Directive Prefix

OMO Slim uses a `[SYSTEM DIRECTIVE: OH-MY-OPENCODE]` prefix for internal messages:

```typescript
// src/shared/system-directive.ts
export const SYSTEM_DIRECTIVE_PREFIX = "[SYSTEM DIRECTIVE: OH-MY-OPENCODE"
```

**Risk**: An adversarial input could include this prefix to impersonate system directives, potentially overriding agent behavior.

**Mitigation**: This is an inherent LLM prompt injection risk. No complete mitigation exists at the plugin level.

### 5.4 Binary Downloads Without Verification

The ast-grep, ripgrep, and comment-checker downloaders fetch binaries from GitHub Releases without hash verification or signature checking.

```typescript
// src/tools/ast-grep/downloader.ts (simplified)
const response = await fetch(downloadUrl, { redirect: 'follow' });
// No checksum verification follows
```

**Risk**: Supply chain attack via compromised GitHub release or CDN MITM.

**Mitigation**: Pre-download binaries on a verified connected machine, compute SHA-256 checksums, verify on the air-gapped machine:

```bash
# On connected machine:
sha256sum ~/.cache/oh-my-opencode-slim/bin/sg > checksums.txt
sha256sum ~/.cache/oh-my-opencode-slim/bin/rg >> checksums.txt

# On air-gapped machine, after transfer:
sha256sum -c checksums.txt
```

## 6. Complete Air-Gap Deployment Checklist

### Pre-Deployment (Connected Machine)

| # | Action | Command / File |
|---|--------|---------------|
| 1 | Install OMO Slim and create tarball | `npm pack oh-my-opencode-slim@latest` |
| 2 | Pre-download ast-grep binary | `bunx oh-my-opencode-slim install` (triggers download), then copy `~/.cache/oh-my-opencode-slim/bin/sg` |
| 3 | Pre-download ripgrep binary | Copy `~/.cache/oh-my-opencode-slim/bin/rg` |
| 4 | Pre-download comment-checker binary | Copy from cache directory |
| 5 | Download schema JSON | `curl -o oh-my-opencode-slim.schema.json https://raw.githubusercontent.com/alvinunreal/oh-my-opencode-slim/master/assets/oh-my-opencode-slim.schema.json` |
| 6 | Verify internal inference server | `curl http://inference.internal/v1/models` returns model list |
| 7 | Compute checksums | `sha256sum` on all downloaded binaries |
| 8 | Package everything | Tarball of npm package + cached binaries + schema |

### Post-Deployment (Air-Gapped Machine)

| # | Action | Verification |
|---|--------|-------------|
| 1 | Install OMO Slim from tarball | `npm install -g ./oh-my-opencode-slim-*.tgz` |
| 2 | Place cached binaries | `ls ~/.cache/oh-my-opencode-slim/bin/{sg,rg}` |
| 3 | Verify binary checksums | `sha256sum -c checksums.txt` |
| 4 | Configure local-only models | Set all agents to `internal/*` in OMO Slim config, pointing to internal inference server |
| 5 | Disable cloud providers | `"disabled_providers": ["anthropic", "openai", "google", "opencode"]` |
| 6 | Disable internet MCPs | `"tools": { "websearch_*": false, "context7_*": false, "grep_app_*": false }` |
| 7 | Set OpenCode env vars | `OPENCODE_DISABLE_MODELS_FETCH=1`, `OPENCODE_DISABLE_LSP_DOWNLOAD=1` |
| 8 | Verify no outbound connections | `ss -tlnp | grep -v 127.0.0.1` should show only local services |
| 9 | Test: `ping 8.8.8.8` fails | Confirm network isolation |
| 10 | Test: `opencode` launches | Verify plugin loads, agents respond |

## 7. Limitations and Open Issues

| Issue | Status | Impact |
|-------|--------|--------|
| No `OPENCODE_OFFLINE=1` toggle | Feature requested ([issue #16117](https://github.com/anomalyco/opencode/issues/16117)) | Must manually disable each network feature |
| No `disabled_hooks` config in OMO Slim | Not in schema | Auto-update checker hook cannot be disabled via config |
| Auto-update fetches npm on every session | By design, 5s timeout | Silent failure in air-gap, minimal latency impact |
| Binary downloads lack hash verification | By design | Must manually verify post-transfer |
| `disabled_mcps` is the only disable mechanism | By design | MCP disabling works; hook disabling does not |
| Librarian agent loses primary capability | By design | No external docs without websearch/context7 |
| Default models point to cloud providers | OpenAI/Kimi defaults | Must override all agent models to internal inference server |
| Skills installer uses `npx` (downloads from GitHub) | By design | Install with `--skills=no` for air-gap |
| No official air-gap deployment guide | Not documented | This document serves as the reference |
| `scoringEngineVersion` config field | Placeholder only | No scoring engine implementation exists in codebase |

## 8. Recommendation

OMO Slim is viable for air-gapped deployment with the mitigations described above. The core orchestration functionality (agent delegation, task management, model routing, session hooks) operates locally and does not require network access. The network dependencies are limited to:

1. **Optional features** (MCPs, auto-updates) — can be disabled
2. **One-time downloads** (binaries) — can be pre-cached
3. **Model inference** — uses internal inference server

The primary risk is supply chain: the plugin downloads and executes unverified binaries. Pre-caching and checksum verification mitigate this.

For high-security environments, consider:
- Auditing the full OMO Slim source code before deployment
- Building from source in a controlled environment
- Monitoring for unexpected process spawning or file access
- Container isolation is optional for air-gapped + local LLM deployments — see the [Security Assessment](./security-assessment.md) for guidance on when containers are warranted vs when configuration hardening alone is sufficient

## Related Documents

- [Architecture Overview](../architecture/overview.md)
- [Security Assessment](./security-assessment.md)
- [OpenCode Setup](../guides/opencode-setup.md)
- [Serena Quickstart](../guides/serena-quickstart.md)
- [Srclight Quickstart](../guides/srclight-quickstart.md)

## References

- [OMO Slim GitHub Repository](https://github.com/alvinunreal/oh-my-opencode-slim)
- [OpenCode Air-Gap Feature Request (issue #16117)](https://github.com/anomalyco/opencode/issues/16117)
- [OpenCode Plugin Documentation](https://opencode.ai/docs/plugins/)
- [OpenCode Agent Configuration](https://opencode.ai/docs/agents/)
- [OpenCode CLI Environment Variables](https://opencode.ai/docs/cli/)
