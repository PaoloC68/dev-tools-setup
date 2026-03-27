# Typical Developer Workflow

A day-in-the-life guide showing how the AI coding stack (OpenCode + Serena + Srclight + Memora)
works together in practice. Every command and prompt here is real — copy-paste ready.

---

## Phase 1: First-Time Setup (Once Per Repo)

### 1.1 Clone and Index

```bash
# Clone the repo
git clone git@internal.git:team/firmware.git
cd firmware

# Generate compile_commands.json (C/C++ — required for Serena's clangd)
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -B build .
ln -sf build/compile_commands.json .

# Create a Srclight workspace (once — skip if workspace already exists)
srclight workspace init myworkspace

# Index for Srclight (keyword + semantic search)
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal \
  srclight index --embed openai:qwen3-embedding-8b

# Add to the workspace
srclight workspace add . -w myworkspace

# Install git hooks (auto-reindex on commit/checkout)
srclight hook install
```

> **How git hooks work with OpenCode**: The hooks are plain shell scripts with the full
> srclight binary path baked in — no PATH dependency. They run in the background (`&` +
> `disown`) after every `git commit` and branch switch, so they don't block OpenCode's
> operations. A `flock` prevents concurrent re-indexes. Output goes to `.srclight/reindex.log`.
>
> **Caveat**: Hooks only update FTS5 indexes (keyword search). They do **not** re-embed —
> semantic search will be stale for new/changed symbols until you manually re-embed:
> ```bash
> OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal \
>   srclight index --embed openai:qwen3-embedding-8b
> ```

### 1.2 Start OpenCode

```bash
opencode
```

First session triggers Serena onboarding automatically:

```
You: Activate the project at /path/to/firmware

Serena: Analyzing project structure...
  Created .serena/project.yml
  Created .serena/memories/
  Found compile_commands.json
  LSP: clangd indexing 847 files...
  Onboarding complete. 12,431 symbols indexed.
```

Generate project context:

```
You: /init
```

Creates `AGENTS.md` — the AI reads this at the start of every session.

### 1.3 Save Initial Context to Memora

```
You: Save this: firmware repo is a real-time control system for the XR-7 actuator.
     Main entry point is src/main.c. Business logic in src/control/.
     Hardware abstraction in src/hal/. Tests in tests/. Build with CMake.

Memora:
  Stored memory #1
  Tags: architecture, onboarding
  Section: project/firmware
```

---

## Phase 2: Start of Day

### 2.1 Resume Context

```
You: What were we working on last session?

Memora (memory_insights):
  Activity summary (last 24h):
  - 5 memories created, 2 TODOs in-progress
  - Focus areas: PID controller refactor, HAL SPI driver

  In-progress:
  #14 — TODO: "Refactor pid_update() to accept config struct instead of 6 params"
  #11 — TODO: "Add timeout handling to spi_transfer()"
```

### 2.2 Orient Yourself

```
You: Show me the codebase overview

→ srclight: codebase_map()

  firmware (C/C++)
  ├── src/control/     — 23 files, 412 symbols (PID, filters, trajectory)
  ├── src/hal/         — 18 files, 287 symbols (SPI, I2C, GPIO, UART)
  ├── src/main.c       — entry point, 45 symbols
  ├── tests/           — 31 files, 198 test functions
  └── include/         — 42 headers

  Total: 847 files, 12,431 symbols, 8,924 edges
```

### 2.3 Check What Changed Overnight

```
You: What changed in the repo since yesterday?

→ srclight: recent_changes(10)

  Recent commits:
  1. a3f21bc (12h ago) — "Fix SPI clock divider for 48MHz" by Alice
  2. 9e8d4a1 (14h ago) — "Add unit test for trajectory planner" by Bob
  3. 2c7f891 (16h ago) — "Update HAL GPIO pin mapping for rev3 board" by Alice
```

### 2.4 Pull and Re-index

```bash
git pull origin main
```

If git hooks are installed, `srclight index` runs automatically in the background.
For embeddings, re-embed manually after significant changes:

```bash
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal \
  srclight index --embed openai:qwen3-embedding-8b
```

Serena picks up changes automatically — no manual step needed.

---

## Phase 3: Understanding Code Before Changing It

### 3.1 Find Code by Concept

```
You: Where do we handle motor position feedback?

→ srclight: hybrid_search("motor position feedback")

  1. src/control/position_loop.c:42 — position_update()      score: 0.94
  2. src/hal/encoder.c:18 — encoder_read_position()           score: 0.89
  3. src/control/pid.c:67 — pid_compute()                     score: 0.78
```

### 3.2 Find Code by Name

```
You: Find the function pid_compute

→ serena: find_symbol("pid_compute")

  pid_compute in src/control/pid.c:67
  Signature: float pid_compute(pid_state_t *state, float setpoint, float measured)
```

### 3.3 Understand Call Relationships

```
You: What calls pid_compute and what does it call?

→ srclight: get_callers("pid_compute", project="firmware")

  Callers:
  1. position_update() in src/control/position_loop.c:58
  2. velocity_loop() in src/control/velocity_loop.c:44
  3. test_pid_step_response() in tests/test_pid.c:23

→ srclight: get_callees("pid_compute", project="firmware")

  Callees:
  1. clamp_output() in src/control/utils.c:12
  2. apply_deadband() in src/control/utils.c:28
```

### 3.4 Blast Radius Before Refactoring

```
You: What breaks if I change pid_compute's signature?

→ srclight: get_dependents("pid_compute", project="firmware", transitive=true)

  Direct callers: 3 symbols
  Transitive dependents: 14 symbols across 6 files

  Files affected:
  - src/control/position_loop.c (2 call sites)
  - src/control/velocity_loop.c (1 call site)
  - src/control/cascade.c (1 call site, via position_update)
  - tests/test_pid.c (3 call sites)
  - tests/test_position.c (2 call sites, via position_update)
  - tests/test_cascade.c (1 call site)
```

### 3.5 Check Who Last Touched This Code

```
You: Who last modified pid_compute and why?

→ srclight: blame_symbol("pid_compute", project="firmware")

  Last modified: 2026-02-14 by Alice
  Commit: d4e2f1a "Refactor PID to support anti-windup"
  Lines 67-112 (45 lines)
```

### 3.6 Find Bug-Prone Files

```
You: Which files change most frequently?

→ srclight: git_hotspots(10, project="firmware")

  1. src/control/pid.c — 28 commits (last 90 days)
  2. src/hal/spi.c — 19 commits
  3. src/control/trajectory.c — 15 commits
```

---

## Phase 4: Making Changes

### 4.1 Navigate to the Right Place

```
You: Show me the pid_compute function

→ serena: find_symbol("pid_compute", include_body=true)

  [Full source code displayed]
```

### 4.2 Understand the Type Hierarchy

```
You: Show me what implements the hal_interface

→ srclight: get_implementors("hal_interface", project="firmware")

  Implementors:
  1. spi_driver in src/hal/spi.c
  2. i2c_driver in src/hal/i2c.c
  3. uart_driver in src/hal/uart.c
  4. mock_driver in tests/mock_hal.c
```

### 4.3 Find Tests for What You're Changing

```
You: What tests cover pid_compute?

→ srclight: get_tests_for("pid_compute", project="firmware")

  Tests:
  1. test_pid_step_response() in tests/test_pid.c:23
  2. test_pid_anti_windup() in tests/test_pid.c:45
  3. test_pid_derivative_kick() in tests/test_pid.c:67
```

### 4.4 Make the Change

```
You: Refactor pid_compute to accept a pid_config_t struct instead of individual parameters.
     Update all callers and tests.

Agent:
  Plan:
  1. Define pid_config_t struct in include/control/pid.h
  2. Modify pid_compute signature
  3. Update 3 callers in src/control/
  4. Update 3 test files
  5. Run tests

  [Implements changes across 7 files]
```

### 4.5 Verify with Serena

```
You: Check for any type errors after the refactor

→ serena: get_diagnostics()

  No errors found. 2 warnings:
  - src/control/pid.c:89: unused variable 'old_output' (from refactor)
  - tests/test_pid.c:34: implicit conversion float→double
```

### 4.6 Rename a Symbol Safely

```
You: Rename pid_compute to pid_controller_step across the entire codebase

→ serena: rename_symbol("pid_compute", new_name="pid_controller_step")

  Renamed in 7 files:
  - src/control/pid.c (definition)
  - src/control/pid.h (declaration)
  - src/control/position_loop.c (call site)
  - src/control/velocity_loop.c (call site)
  - src/control/cascade.c (call site)
  - tests/test_pid.c (3 call sites)
  - tests/test_position.c (2 call sites)
```

---

## Phase 5: Record Decisions

### 5.1 Save Architectural Decision

```
You: Remember this: We changed pid_compute to accept a pid_config_t struct
     instead of 6 individual parameters. This simplifies the API and makes it
     easier to add new PID features without changing every caller.
     The struct is defined in include/control/pid.h.

Memora:
  Stored memory #15
  Tags: architecture, refactor, pid
  Section: decisions/control
```

### 5.2 Track a Bug for Later

```
You: Create an issue: The SPI driver doesn't handle bus contention when two
     peripherals try to communicate simultaneously. Severity: major.
     Component: hal/spi.

Memora:
  Created issue #16
  Status: open | Severity: major | Component: hal/spi
  Tags: bug, spi, hal
```

### 5.3 Link Related Decisions

```
You: Link memory #15 (PID config struct) to memory #8 (control API design principles).
     Edge type: implements.

Memora:
  Linked #15 → #8 (implements)
  Knowledge graph updated.
```

---

## Phase 6: Multi-Repo Work

### 6.1 Add Another Repo to the Workspace

```bash
git clone git@internal.git:team/bootloader.git
cd bootloader

# Generate compile_commands.json
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -B build .
ln -sf build/compile_commands.json .

# Add to existing workspace and index
srclight workspace add . -w myworkspace
srclight workspace index -w myworkspace -p bootloader \
  --embed openai:qwen3-embedding-8b

# Install hooks
srclight hook install
```

The running MCP server picks up the new project automatically — no restart needed.

### 6.2 Search Across All Repos

```
You: Find all CRC calculation implementations across all repos

→ srclight: hybrid_search("CRC calculation")

  firmware:
  1. src/util/crc32.c:15 — crc32_compute()
  2. src/hal/flash.c:89 — verify_flash_crc()

  bootloader:
  1. src/integrity.c:22 — boot_check_crc()
  2. src/update.c:156 — validate_firmware_image()
```

### 6.3 Cross-Repo Dependency Analysis

```
You: Does the bootloader use any functions from the firmware repo?

→ srclight: search_symbols("firmware", project="bootloader")
→ srclight: hybrid_search("shared header firmware bootloader")

  Found shared headers:
  - include/shared/protocol.h (both repos include this)
  - include/shared/crc.h (identical CRC implementation)
```

### 6.4 Switch Serena's Active Project

Serena works on one project at a time. Switch when you change focus:

```
You: Activate the project at /path/to/bootloader

Serena:
  Deactivated: firmware
  Activated: bootloader
  LSP: clangd indexing 234 files...
  Ready. 3,821 symbols indexed.
```

Srclight doesn't need switching — it always searches all projects, filtered by the
`project` parameter when needed.

---

## Phase 7: Code Review

### 7.1 Review Incoming Changes

```
You: What's changed in my working directory?

→ srclight: whats_changed(project="firmware")

  Modified:
  - src/control/pid.c (+45, -23)
  - src/control/pid.h (+12, -3)
  - tests/test_pid.c (+67, -34)

  New:
  - include/control/pid_types.h
```

### 7.2 Check for Common Issues

```
You: Review the changes in src/control/pid.c for potential issues

Agent:
  [Reads the diff, checks against Memora for relevant decisions]

  Observations:
  1. The new pid_config_t struct has 8 fields — consider grouping into sub-structs
     (e.g. gains, limits, filtering) if it grows further
  2. The anti_windup_limit field defaults to 0.0 — should this be INFINITY instead?
  3. Good: all callers updated, no orphaned references
```

### 7.3 Verify Test Coverage

```
You: Are there tests for every function I changed?

→ srclight: get_tests_for("pid_controller_step", project="firmware")
→ srclight: get_tests_for("pid_config_init", project="firmware")

  pid_controller_step: 3 tests ✓
  pid_config_init: 0 tests ✗ — needs test coverage
```

---

## Phase 8: End of Day

### 8.1 Check Open Items

```
You: What open TODOs and issues do we have?

Memora (memory_list, filter=status:open):
  TODOs:
  #14 — in-progress: "Refactor pid_update() to accept config struct" ← done, mark complete
  #11 — in-progress: "Add timeout handling to spi_transfer()"

  Issues:
  #16 — open/major: "SPI bus contention not handled"
  #9 — open/minor: "UART ring buffer wastes 1 byte"
```

### 8.2 Update TODO Status

```
You: Mark TODO #14 as completed.

Memora:
  Updated #14: in-progress → completed
```

### 8.3 Save Session Summary

```
You: Save this session summary: Completed PID refactor (pid_compute → pid_controller_step,
     now takes pid_config_t struct). All callers updated, all tests pass. Next: add tests
     for pid_config_init, then start on SPI timeout handling (#11).

Memora:
  Stored memory #17
  Tags: session-summary, progress
  Section: sessions/2026-03-25
```

### 8.4 Commit

```
You: Commit the PID refactor changes

Agent:
  [Creates atomic commits following project conventions]

  Created 2 commits:
  1. "Add pid_config_t struct, refactor pid_controller_step to use it"
     — include/control/pid_types.h, src/control/pid.h, src/control/pid.c
  2. "Update callers and tests for pid_controller_step refactor"
     — src/control/position_loop.c, velocity_loop.c, cascade.c, tests/test_pid.c
```

Git hooks trigger `srclight index` in the background — FTS5 indexes are fresh for the
next session. Semantic embeddings update if the `--embed` flag was set in the hook config.

---

## Common Prompts Cheat Sheet

### Orientation

| Prompt | What happens |
|--------|-------------|
| "Show me the codebase overview" | `srclight: codebase_map()` |
| "What changed recently?" | `srclight: recent_changes()` |
| "What were we working on?" | `memora: memory_insights()` |
| "Which files change most?" | `srclight: git_hotspots()` |

### Finding Code

| Prompt | What happens |
|--------|-------------|
| "Where is authentication handled?" | `srclight: hybrid_search("authentication")` |
| "Find function `validate_token`" | `serena: find_symbol("validate_token")` |
| "Show me the source of `parse_config`" | `serena: find_symbol("parse_config", include_body=true)` |
| "What implements the `Driver` interface?" | `srclight: get_implementors("Driver")` |

### Understanding Relationships

| Prompt | What happens |
|--------|-------------|
| "What calls `pid_compute`?" | `srclight: get_callers("pid_compute")` |
| "What does `init_system` call?" | `srclight: get_callees("init_system")` |
| "What breaks if I change `spi_transfer`?" | `srclight: get_dependents("spi_transfer", transitive=true)` |
| "Show the class hierarchy for `Sensor`" | `srclight: get_type_hierarchy("Sensor")` |

### Git Intelligence

| Prompt | What happens |
|--------|-------------|
| "Who last changed `main.c`?" | `srclight: blame_symbol("main")` |
| "What's uncommitted?" | `srclight: whats_changed()` |
| "History of `spi_transfer`" | `srclight: changes_to("spi_transfer")` |

### Refactoring

| Prompt | What happens |
|--------|-------------|
| "Rename `old_name` to `new_name`" | `serena: rename_symbol("old_name", new_name="new_name")` |
| "Find all tests for `my_function`" | `srclight: get_tests_for("my_function")` |
| "Check for errors after changes" | `serena: get_diagnostics()` |

### Memory and Decisions

| Prompt | What happens |
|--------|-------------|
| "Remember: we chose X because Y" | `memora: memory_create(content="...", tags=[...])` |
| "What did we decide about auth?" | `memora: memory_search("auth")` |
| "What open issues exist?" | `memora: memory_list(filter=status:open)` |
| "Mark TODO #5 complete" | `memora: memory_update(id=5, status="completed")` |

---

## Quick Reference: CLI Commands

### Daily

```bash
# Pull and check status
git pull origin main
srclight workspace status -w myworkspace

# Start OpenCode
opencode

# Re-embed after major changes (manual — hooks only update FTS5)
OPENAI_API_KEY=sk-xxx OPENAI_BASE_URL=http://inference.internal \
  srclight workspace index -w myworkspace --embed openai:qwen3-embedding-8b
```

### Adding a New Repo

```bash
cd /path/to/new-repo
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON -B build .   # C/C++ only
ln -sf build/compile_commands.json .                    # C/C++ only

srclight workspace add . -w myworkspace
srclight workspace index -w myworkspace -p new-repo \
  --embed openai:qwen3-embedding-8b
srclight hook install
```

Then in OpenCode:

```
You: Activate the project at /path/to/new-repo
```

### Workspace Management

```bash
srclight workspace list                             # list all workspaces
srclight workspace status -w myworkspace            # show all repos with stats
srclight workspace remove old-repo -w myworkspace   # remove a repo
srclight hook status --workspace myworkspace         # check hook status
```

---

## Next Steps

- [Architecture Overview](../architecture/overview.md)
- [Serena Quickstart](./serena-quickstart.md) — symbolic navigation and refactoring
- [Srclight Quickstart](./srclight-quickstart.md) — semantic search and git intelligence
- [Memora Quickstart](./memora-quickstart.md) — cross-session memory and decisions
- [OpenCode Quickstart](./opencode-quickstart.md) — base agent configuration
- [OpenCode Setup](./opencode-setup.md) — multi-agent orchestration with OMO Slim