# Security Assessment: Air-Gapped AI Coding Architecture

## Executive Summary

This assessment covers the security posture of the air-gapped AI coding architecture, a 100% offline RAG system for C/C++ codebase intelligence. The stack consists of OpenCode (base agent), Oh-My-OpenCode-Slim (orchestration plugin), and three MCP servers (Serena, Srclight, Memora).

The air-gapped design eliminates most network-based attack vectors. However, several component-level risks remain, particularly around Serena's shell execution capabilities, unauthenticated service endpoints, and supply chain dependencies that assume network access during initial setup.

**Severity Summary:**

| Severity | Count | Key Areas |
|----------|-------|-----------|
| CRITICAL | 1 | Serena shell command execution |
| HIGH | 3 | Unauthenticated services, filesystem traversal, MCP prompt injection |
| MEDIUM | 4 | Supply chain (npm/pip), model downloads, Memora cloud sync, cross-agent abuse |
| LOW | 2 | Local SQLite access, memory file permissions |

## Threat Model

### What the Stack Protects

- Proprietary C/C++ source code (the primary asset)
- Architectural decisions and session history stored in Memora
- Embedding vectors that encode semantic meaning of source code
- Development workflow integrity (no unauthorized code modifications)

### Threat Actors

| Actor | Motivation | Attack Surface |
|-------|-----------|----------------|
| Supply chain attacker | Inject malicious code via compromised packages | npm, pip, Hugging Face Hub, Ollama registry |
| Insider with local access | Exfiltrate code or tamper with codebase | MCP tools, Serena shell access, filesystem |
| Compromised MCP server | Hijack agent behavior or exfiltrate data | Tool descriptions, prompt injection, stdio transport |
| Malicious prompt crafter | Trick agents into executing harmful operations | OpenCode agent inputs, MCP tool responses |

### Out of Scope

- Physical security of the air-gapped machine
- Operating system kernel vulnerabilities
- Hardware-level attacks (side channels, firmware)

## Component-Level Assessment

### 1. OpenCode (Orchestration Layer)

OpenCode routes queries to MCP servers and coordinates six specialized agents (architect, implementer, reviewer, debugger, researcher, security).

**Findings:**

| ID | Severity | Finding |
|----|----------|---------|
| OC-1 | MEDIUM | Agent prompts can be manipulated via crafted MCP tool responses, causing unintended actions |
| OC-2 | MEDIUM | Cross-agent context sharing lacks isolation boundaries. A compromised agent response can influence subsequent agent decisions |
| OC-3 | LOW | Configuration stored in `.opencode/` directory with no integrity verification |

**Risk:** OpenCode trusts MCP tool outputs without validation. A poisoned tool response from any spoke can influence the orchestrator's behavior across all agents.

**Mitigations:**

- Restrict agent permissions to minimum required tools
- Validate MCP responses before passing to agents
- Monitor agent actions for anomalous patterns (e.g., unexpected file modifications)

### 2. Serena (Symbolic Layer)

Serena provides LSP-based code navigation and editing via clangd/ccls. It exposes the most dangerous attack surface in the stack.

**Findings:**

| ID | Severity | Finding |
|----|----------|---------|
| SR-1 | CRITICAL | `ExecuteShellCommandTool` runs arbitrary commands via `subprocess.Popen` with `shell=True` and no sandboxing. Any prompt injection that reaches this tool gains full shell access. |
| SR-2 | HIGH | Dashboard binds to `0.0.0.0:24282` by default. MCP server binds to `0.0.0.0:9121`. Both are unauthenticated, exposing full tool access to any host on the local network. |
| SR-3 | HIGH | `file_tools.py` provides unrestricted filesystem read/write. No path validation or chroot. Path traversal (e.g., `../../etc/passwd`) is possible. |
| SR-4 | MEDIUM | Memory files in `.serena/memories/` are plain markdown with no access controls or integrity checks. |

**Source:** [Serena security audit discussion](https://github.com/oraios/serena/discussions/380)

**Mitigations:**

Serena's built-in configuration provides effective hardening without container isolation. The `ide-assistant` and `claude-code` contexts already exclude `execute_shell_command` by default.

**Primary mitigation (configuration hardening):**

```yaml
# .serena/project.yml - hardened for air-gapped deployment
read_only: false                    # Enable editing (needed for coding)
enable_gui_logging: false           # Disable dashboard (no 0.0.0.0 binding)

excluded_tools:
  - execute_shell_command           # Explicitly remove shell access
```

This eliminates the two most critical risks (SR-1 shell execution and SR-2 network binding). File system access (SR-3) remains, but in an air-gapped environment with a local LLM, there is no exfiltration path — data read into the LLM context stays on-device.

**When container isolation IS warranted:**

| Scenario | Why Container Matters |
|----------|----------------------|
| Remote LLM provider (cloud API) | File contents read by Serena travel to cloud — real exfiltration risk |
| Multi-user shared server | Other users could exploit Serena's process |
| `execute_shell_command` enabled (for tests/builds) | Container limits blast radius of prompt-injected commands |
| Regulatory/compliance requirements | Defense-in-depth may be mandated regardless of actual risk |

**Container isolation (when needed):**

```bash
docker run --network=none \
  --read-only \
  --tmpfs /tmp:noexec,nosuid \
  --cap-drop=ALL \
  -v /path/to/project:/workspace:rw \
  -v /usr/lib/llvm/bin/clangd:/usr/bin/clangd:ro \
  serena-image
```

Note: Container deployment adds complexity — volume mounts for project files, LSP server binaries, and system headers must be configured correctly for clangd/ccls to function.

### 3. Srclight (Semantic Layer)

Srclight indexes code into a local SQLite database using hybrid search (FTS5 trigram + semantic embeddings). The default embedding model is `qwen3-embedding` via Ollama, with `nomic-embed-text` as a lighter alternative.

**Findings:**

| ID | Severity | Finding |
|----|----------|---------|
| SL-1 | MEDIUM | Ollama attempts to pull models from its registry when a model is not found locally. If the air-gap has a leak, this silently downloads data. |
| SL-2 | MEDIUM | If `sentence-transformers` is used as an alternative provider, it auto-downloads models from Hugging Face Hub on first use. |
| SL-3 | LOW | SQLite database (`.srclight/index.db`) stores embedding vectors that encode semantic meaning of source code. No encryption at rest. |

**Mitigations:**

```bash
# Pre-download Ollama embedding model BEFORE going air-gapped
ollama pull qwen3-embedding
# Or lighter alternative:
ollama pull nomic-embed-text

# If using sentence-transformers as an alternative provider:
export HF_HUB_OFFLINE=1
export TRANSFORMERS_OFFLINE=1
```

```yaml
# .srclight/config.yml - enforce offline operation
offline_mode: true
```

- Transfer pre-downloaded models via secure removable media
- Verify model checksums after transfer
- Set `offline_mode: true` in Srclight configuration

### 4. Memora (Persistence Layer)

Memora provides cross-session memory using a local SQLite backend. It has optional cloud sync features that present a direct air-gap violation risk.

**Findings:**

| ID | Severity | Finding |
|----|----------|---------|
| ME-1 | HIGH | Memora supports OPTIONAL cloud sync to S3, Cloudflare R2, and Cloudflare D1. If enabled (even accidentally), session data, architectural decisions, and code context leak to external services. |
| ME-2 | MEDIUM | Environment variables `MEMORA_STORAGE_URI` and `CLOUDFLARE_API_TOKEN` trigger cloud sync when set. A misconfigured `.env` file or inherited shell environment can silently enable this. |
| ME-3 | LOW | SQLite database (`.memora/memory.db`) stores session history and embeddings without encryption. Accessible to any local user with file permissions. |

**Mitigations:**

```bash
# Verify cloud sync is NOT configured
env | grep -i "MEMORA_STORAGE\|CLOUDFLARE\|AWS_\|S3_"
# Expected output: empty. Any matches indicate a misconfiguration.

# Explicitly unset in shell profile
unset MEMORA_STORAGE_URI
unset CLOUDFLARE_API_TOKEN
unset CLOUDFLARE_ACCOUNT_ID
```

- Only use local SQLite mode. Never configure cloud storage backends.
- Audit environment variables on every session start.
- Block outbound network at the firewall level as a defense-in-depth measure.
- Restrict file permissions on `.memora/memory.db` to the development user only.

### 5. MCP Protocol (Communication Layer)

MCP connects OpenCode to its spoke servers (Serena, Srclight, Memora) via stdio or HTTP transports. The protocol itself introduces several attack vectors.

**Findings:**

| ID | Severity | Finding |
|----|----------|---------|
| MC-1 | HIGH | **Prompt injection via tool responses.** A compromised MCP server can embed instructions in tool output that manipulate the LLM's behavior. The LLM cannot reliably distinguish tool data from tool instructions. |
| MC-2 | HIGH | **Tool poisoning.** Malicious tool descriptions can hijack agent behavior by embedding hidden instructions in the tool's description field, which the LLM reads to decide how to use the tool. |
| MC-3 | MEDIUM | **Rug pull attacks.** An MCP server can change tool behavior after initial trust is established. Version pinning mitigates this, but runtime behavior changes within a pinned version are still possible. |
| MC-4 | MEDIUM | **Data exfiltration via stdio.** The stdio transport gives MCP servers shell-like access to the host process's I/O. A malicious server can read environment variables, inspect process state, or write to unexpected file descriptors. |
| MC-5 | MEDIUM | **Supply chain attacks.** Compromised MCP server packages (via npm or pip) execute with full user privileges on installation. |

**Mitigations:**

```bash
# Audit MCP server security with mcp-scan
npx @anthropic/mcp-scan

# Pin MCP server versions in configuration
# BAD:  npx -y @oraios/serena
# GOOD: npx @oraios/serena@1.2.3
```

- Pin all MCP server versions. Never use `latest` or `-y` flag in production.
- Audit tool descriptions for hidden instructions before trusting a new MCP server.
- Use allowlists to restrict which tools each agent can invoke.
- Monitor MCP server behavior for changes between sessions.
- Prefer stdio transport over HTTP to avoid network exposure.

## Supply Chain Risks

### NPX Execution

The `npx -y` pattern used in many MCP server configurations always downloads packages from the npm registry unless locally cached. This is incompatible with air-gapped operation.

```bash
# BROKEN in air-gapped environments:
npx -y @oraios/serena

# CORRECT: pre-install globally before going air-gapped
npm install -g @oraios/serena@1.2.3

# ALTERNATIVE: set up a local npm registry mirror
# Install Verdaccio on the air-gapped network
npm install -g verdaccio
verdaccio &
npm set registry http://localhost:4873
```

### Python Dependencies

```bash
# Pre-download all pip packages
pip download -d ./offline-packages -r requirements.txt

# Install from local cache in air-gapped environment
pip install --no-index --find-links=./offline-packages -r requirements.txt
```

### Model Downloads

| Model | Source | Pre-Download Command |
|-------|--------|---------------------|
| qwen3-embedding (default) | Ollama Registry | `ollama pull qwen3-embedding` |
| nomic-embed-text (lighter) | Ollama Registry | `ollama pull nomic-embed-text` |
| clangd | LLVM releases | Download binary from llvm.org, transfer via secure media |

All downloads must happen on a network-connected staging machine, then transfer to the air-gapped environment via verified removable media.

## Air-Gap Compliance Checklist

Run this checklist before declaring the environment air-gap compliant.

| # | Component | Check | Command / Action | Pass |
|---|-----------|-------|------------------|------|
| 1 | Network | No outbound connections possible | `ping 8.8.8.8` should fail; verify firewall rules block all egress | [ ] |
| 2 | Network | All services bound to localhost | `ss -tlnp \| grep -v 127.0.0.1` should return empty | [ ] |
| 3 | Srclight | Embedding model pre-downloaded | `ls ~/.cache/torch/sentence_transformers/` contains model directory | [ ] |
| 4 | Srclight | Offline mode enforced | `grep offline_mode .srclight/config.yml` returns `true` | [ ] |
| 5 | Srclight | HF offline env var set | `echo $HF_HUB_OFFLINE` returns `1` | [ ] |
| 6 | Ollama | Embedding model pre-pulled | `ollama list` shows required model | [ ] |
| 7 | npm | Packages pre-installed (no npx -y) | `which serena` resolves to local install | [ ] |
| 8 | npm | Registry unreachable or local | `npm ping` fails or points to localhost | [ ] |
| 9 | pip | No index access | `pip install --dry-run requests` fails | [ ] |
| 10 | Memora | Cloud sync disabled | `env \| grep -i "MEMORA_STORAGE\|CLOUDFLARE"` returns empty | [ ] |
| 11 | Memora | Only SQLite backend active | Verify `.memora/memory.db` exists, no S3/R2 config present | [ ] |
| 12 | Serena | Dashboard disabled or localhost-only | `ss -tlnp \| grep 24282` shows `127.0.0.1` or nothing | [ ] |
| 13 | Serena | MCP server localhost-only | `ss -tlnp \| grep 9121` shows `127.0.0.1` or nothing | [ ] |
| 14 | Serena | Read-only mode (if applicable) | `grep read_only .serena/project.yml` returns `true` | [ ] |
| 15 | DNS | No external resolution | `nslookup google.com` fails | [ ] |
| 16 | Firewall | Egress blocked | `iptables -L OUTPUT` or equivalent shows DROP/REJECT default | [ ] |

## Hardening Recommendations

### Priority 1: Configuration Hardening (All Deployments)

Apply these settings to eliminate the highest-risk attack surfaces without container overhead:

```yaml
# .serena/project.yml
enable_gui_logging: false           # Disables dashboard (no 0.0.0.0 binding)

excluded_tools:
  - execute_shell_command           # Removes arbitrary shell execution
```

The `ide-assistant` and `claude-code` contexts already exclude `execute_shell_command` by default. Adding it to `excluded_tools` in `project.yml` enforces this regardless of context.

In an air-gapped environment with a local LLM (Ollama), configuration hardening alone is sufficient: there is no network to exfiltrate data to, and the LLM runs on-device.

### Priority 2: Container Isolation (When Required)

Container isolation provides additional protection when `execute_shell_command` is enabled (for running tests/builds), when using a remote LLM provider, or when compliance mandates defense-in-depth.

```bash
# Serena in a locked-down container (only when needed)
docker run \
  --network=none \
  --read-only \
  --tmpfs /tmp:noexec,nosuid \
  --cap-drop=ALL \
  -v /path/to/project:/workspace:rw \
  -v /usr/lib/llvm/bin/clangd:/usr/bin/clangd:ro \
  serena-image
```

Note: Container deployment requires mounting project files, LSP binaries, and system headers for clangd/ccls to function. This adds deployment complexity that is not justified when configuration hardening alone addresses the threat model.

### Priority 3: Service Binding

Bind all services to `127.0.0.1` exclusively. Never use `0.0.0.0`.

```bash
# Verify no services listen on all interfaces
ss -tlnp | grep "0.0.0.0"
# Expected: empty output
```

### Priority 3: Dependency Pinning

Pin every dependency version. Use lockfiles.

```bash
# npm: use package-lock.json, never --save with ranges
npm ci --ignore-scripts

# pip: use pip-compile for deterministic installs
pip-compile requirements.in --generate-hashes
pip install --require-hashes -r requirements.txt
```

### Priority 4: Serena Hardening

```yaml
# .serena/project.yml
read_only: true                # Unless active editing is required
enable_gui_logging: false      # Disable dashboard entirely
```

- Disable `ExecuteShellCommandTool` if shell access is not needed
- Apply AppArmor or SELinux profiles to restrict filesystem access
- Use bind mounts to limit Serena's view to the project directory only

### Priority 5: Environment Variables

```bash
# Add to shell profile (.bashrc, .zshrc)
export HF_HUB_OFFLINE=1
export TRANSFORMERS_OFFLINE=1
unset MEMORA_STORAGE_URI
unset CLOUDFLARE_API_TOKEN
unset CLOUDFLARE_ACCOUNT_ID
```

### Priority 6: MCP Server Auditing

```bash
# Scan MCP servers for known vulnerabilities
npx @anthropic/mcp-scan

# Review tool descriptions for hidden instructions
# Manually inspect each MCP server's tool list before first use
```

- Pin MCP server versions in all configuration files
- Audit tool descriptions for prompt injection payloads
- Use tool allowlists to restrict agent access to approved tools only
- Monitor for behavioral changes between MCP server updates

### Priority 7: Monitoring

```bash
# Watch for unexpected outbound connection attempts
# (should all fail in a properly air-gapped environment)
sudo tcpdump -i any 'not host 127.0.0.1 and not host ::1' -c 100

# Monitor file access patterns
inotifywait -m -r /path/to/project --format '%T %w%f %e' --timefmt '%H:%M:%S'
```

## Related Documents

- [Architecture Overview](../architecture/overview.md)
- [Serena Quickstart](../guides/serena-quickstart.md)
- [Serena LSP Best Practices](serena-lsp-guide.md)
- [Srclight Setup](../guides/srclight-setup.md)
- [Srclight Quickstart](../guides/srclight-quickstart.md)
- [Memora Configuration](../guides/memora-config.md)
- [OpenCode Setup](../guides/opencode-setup.md)
- [OpenCode Quickstart](../guides/opencode-quickstart.md)
