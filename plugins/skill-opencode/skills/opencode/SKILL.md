---
name: opencode
description: Run opencode CLI to build, analyze, or refactor code using cloud models (nemotron, gpt-5.x, claude, gemini). Scaffolds projects, dispatches builds, resumes sessions, and critically reviews output.
---

# OpenCode Skill Guide

## Two Modes

### Mode 1: Run (direct task dispatch)
Run opencode non-interactively from Claude — like Codex `exec`. Use for builds, refactors, analysis.

### Mode 2: Scaffold + Run (new project builds)
Claude designs SPEC.md + CLAUDE.md, then dispatches opencode to build it. The proven "spec-driven build" pattern.

---

## Running a Task

1. Ask the user (via `AskUserQuestion`) which model to run AND which reasoning variant in a **single prompt with two questions**.
2. Assemble the command with appropriate options.
3. Run the command, capture output, and summarize the outcome.
4. **After opencode completes**, inform the user: "You can resume this session at any time by saying 'opencode resume'."

### CLI Reference

```bash
opencode run [message..] \
  -m, --model <provider/model>     # model in provider/model format
  --variant <effort>               # reasoning effort: high, max, minimal (provider-specific)
  --dir <DIR>                      # directory to run in
  -c, --continue                   # continue the last session
  -s, --session <id>               # continue a specific session
  --fork                           # fork session before continuing
  --format <default|json>          # output format
  -f, --file <path>                # attach file(s) to message
  --title <title>                  # session title
  --thinking                       # show thinking blocks (default: false)
```

### Command Assembly

```bash
# Standard build task
opencode run --dir /path/to/project -m opencode/nemotron-3-ultra-free \
  "Read SPEC.md and CLAUDE.md. Build everything." 2>/dev/null

# With model + variant
opencode run --dir /path/to/project -m opencode/gpt-5.6-terra --variant high \
  "your prompt here" 2>/dev/null

# Resume last session
opencode run -c "follow-up instructions here" 2>/dev/null

# Resume specific session
opencode run -s <session-id> "follow-up instructions here" 2>/dev/null
```

### Available Models (key ones)

> **This table is a floor, not a ceiling — snapshot as of 2026-08.** Model knowledge
> goes stale in weeks, and a stale table is worse than none: the agent confidently
> reaches for a model that no longer exists. Run `opencode models` and check the
> provider's own list before choosing. Entries here that have since been retired
> should be replaced, not worked around.

| Model | Provider | Best For | Free? |
|-------|----------|----------|-------|
| `opencode/nemotron-3-ultra-free` | NVIDIA | Rust/Python builds, free tier | Yes |
| `opencode/gpt-5.6-sol` | OpenAI | Hard, ambiguous, high-value work | No |
| `opencode/gpt-5.6-terra` | OpenAI | Workhorse — complex multi-file projects | No |
| `opencode/gpt-5.6-luna` | OpenAI | Fast / cheap, high-volume work | No |
| `opencode/gpt-5.3-codex-spark` | OpenAI | Fast code generation | No |
| `opencode/claude-opus-5` | Anthropic | Highest quality reasoning | No |
| `opencode/claude-sonnet-5` | Anthropic | Good balance speed/quality | No |
| `opencode/claude-haiku-4-5` | Anthropic | Cheap subagents, high volume | No |
| `opencode/gemini-3.6-flash` | Google | Fast, large context tasks | No |

⚠️ `opencode/gpt-5.4` and `gpt-5.4-mini` are retiring — prefer the `5.6` tier.

Run `opencode models` for the full, current list — it is authoritative over the table above.

**Listed is not the same as selectable.** `opencode models` prints the whole registry, including models your account, plan, or provider credentials can't actually reach. If a dispatch fails on model selection, that's the reason — drop to the nearest tier that works rather than retrying the same slug.

### Variant (Reasoning Effort)

Provider-specific reasoning control, equivalent to Codex's `model_reasoning_effort`:

| Variant | Meaning |
|---------|---------|
| `max` | Maximum reasoning (xhigh equivalent) |
| `high` | Strong reasoning |
| (omitted) | Default/medium |
| `minimal` | Fast, less reasoning |

### Quick Reference

| Use case | Command pattern |
|----------|----------------|
| Build from spec | `opencode run --dir <DIR> -m <model> "Read SPEC.md and build everything" 2>/dev/null` |
| Quick edit | `opencode run --dir <DIR> -m opencode/gpt-5.6-luna "fix the bug in src/main.rs" 2>/dev/null` |
| Analysis (read-only) | `opencode run --dir <DIR> -m <model> "analyze this codebase for X" 2>/dev/null` |
| Resume | `opencode run -c "new instructions" 2>/dev/null` |
| Resume specific | `opencode run -s <session-id> "new instructions" 2>/dev/null` |
| Free tier build | `opencode run --dir <DIR> -m opencode/nemotron-3-ultra-free "prompt" 2>/dev/null` |
| Attach file | `opencode run -f SPEC.md -m <model> "build this" 2>/dev/null` |

---

## Scaffolding a New Project (Mode 2)

When the user needs a new standalone tool built, scaffold it first:

### Arguments

```
/opencode <language> <project-name> "<one-line description>"
/opencode build "<prompt>"                                     # direct run, no scaffold
/opencode resume                                               # resume last session
/opencode resume "<follow-up>"                                 # resume with new instructions
```

Examples:
```
/opencode rust selfimprove-harness "crash-resilient loop execution engine"
/opencode python gold-validator "validate gold JSON files against schema"
/opencode build "Read SPEC.md and implement all features"
/opencode resume "the tests are failing, fix the config parsing"
```

### Scaffold Steps

1. **Create project directory** at `~/Dev/<project-name>/`
   - Rust: `cargo init`
   - Python: create `pyproject.toml` + `src/`
   - Node: `npm init -y`

2. **Interview the user** — ask 3-5 questions about requirements via `AskUserQuestion`:
   - What inputs does it take?
   - What outputs does it produce?
   - What error conditions should it handle?
   - Any config file needed?
   - What systems does it integrate with? (env vars, APIs, file paths)

3. **Write SPEC.md** — full requirements doc with:
   - Purpose and usage
   - Inputs/outputs
   - Behavior per feature (numbered, detailed)
   - Config format with example (if any)
   - Environment variables it reads
   - Error handling and exit codes
   - Test cases (unit + integration)
   - Constraints (dependencies, platform, no-unsafe, etc.)
   - Build & run commands

4. **Write CLAUDE.md** — build instructions for the AI coder:
   - "Read SPEC.md and build everything"
   - Module architecture (one file per concern)
   - Build command
   - Dependency list with versions
   - Key constraints the builder should know

5. **Write config file** — pre-populated with real values from the current environment

6. **Pre-populate dependencies** — for Rust: fill `Cargo.toml [dependencies]`; for Python: fill `pyproject.toml`; for Node: run `npm install` with needed packages

7. **Dispatch the build:**
   ```bash
   opencode run --dir ~/Dev/<project-name> -m <model> --variant high \
     "Read SPEC.md and CLAUDE.md thoroughly. Build everything described in the spec — all features, all modules, all tests. Make sure the build succeeds and tests pass." 2>/dev/null
   ```

8. **After build completes** — Claude reviews, tests, fixes integration bugs, commits

### Phased Build Pattern (recommended for 5+ module projects)

One big spec overwhelms the model — cross-module wiring breaks. Instead, split into phases with verification gates between each. Each phase is a separate `opencode run` call. Claude verifies between phases.

**Phase template:**

| Phase | Scope | Gate |
|-------|-------|------|
| 1. Core types | Data models, config, dependencies | Compiles |
| 2. Logic | Business logic modules + unit tests | Tests pass |
| 3. I/O | File readers, HTTP, subprocess calls | Tests pass |
| 4. UI / CLI | Entry point, user-facing layer | Full build succeeds |
| 5. Integration | Wire everything, lib.rs/index.ts, main | Full test suite + manual run |

**Language-specific gates:**

| Gate | Rust | Python | Node/TS |
|------|------|--------|---------|
| Compiles | `cargo build` | `python -c "import pkg"` | `tsc --noEmit` |
| Tests pass | `cargo test` | `pytest` | `npm test` |
| Full build | `cargo build --release` | `pip install -e .` | `npm run build` |

**How to run phased:**

```bash
# Phase 1: types + config
opencode run --dir <DIR> -m <model> "Read SPEC.md sections 'Data Model' and 'Configuration'. \
  Build data.rs and config.rs only. Make cargo build succeed." 2>/dev/null

# Claude verifies: cargo build ✓

# Phase 2: readers + tests
opencode run -c "Now build reader.rs and status.rs per the spec. \
  Add unit tests for parsing. Make cargo test pass." 2>/dev/null

# Claude verifies: cargo test ✓ (N tests passing)

# Phase 3-5: continue pattern...
```

**Why this works:** Models excel at module-internal logic but drop cross-module contracts (env vars, lifecycle files, directory creation). Catching errors at phase boundaries prevents cascading breakage. Matches nemotron-3-super's design strength: agentic multi-step workflows.

---

## Following Up / Resuming

- After every opencode command, inform the user they can resume.
- When resuming, use `-c` for last session or `-s <id>` for a specific one:
  ```bash
  opencode run -c "The tests are failing because X. Fix it and re-run cargo test." 2>/dev/null
  ```
- List sessions with: `opencode session list`
- Restate the model when proposing follow-up actions.

---

## Critical Evaluation of OpenCode Output

OpenCode dispatches to various AI models. Treat output as a **colleague's work, not authoritative**.

### Guidelines
- **Trust your own knowledge** when confident. Push back on incorrect claims.
- **Research disagreements** using WebSearch or docs before accepting.
- **Remember knowledge cutoffs** — models may not know about recent releases.
- **Don't defer blindly** — evaluate suggestions critically.

### Known Integration Gaps (from experience)

OpenCode-built projects consistently miss these — **always verify after build**:

1. **Environment variable plumbing** — sets vars to wrong values (URLs instead of enum strings), misses vars that downstream scripts need
2. **Lifecycle management** — forgets to create PID files, heartbeat files, or lock files that its own code checks
3. **Directory creation** — assumes output dirs exist without `mkdir -p`
4. **Cross-component contracts** — misses env vars, file paths, or API endpoints that other scripts in the ecosystem depend on
5. **Config field naming** — serde field names may not match TOML keys

### Post-Build Checklist

After every opencode build, run through:

- [ ] `cargo build --release` / `npm run build` / `python -m py_compile` — does it compile?
- [ ] `cargo test` / `npm test` / `pytest` — do tests pass?
- [ ] Run against live infrastructure — does it actually work end-to-end?
- [ ] Check env var passthrough — are all needed vars forwarded to subprocesses?
- [ ] Check file/dir creation — does it create dirs before writing to them?
- [ ] Check lifecycle — PID files, lock files, signal handlers, cleanup on exit?
- [ ] Check backward compatibility — does output match expected formats?

### When OpenCode is Wrong

1. State the issue clearly to the user
2. Fix it yourself (Claude is better at integration-level fixes)
3. Optionally resume the session to ask for targeted fixes:
   ```bash
   opencode run -c "The PID file is never written. Add fs::write of PID on startup in lib.rs and cleanup on exit." 2>/dev/null
   ```
4. Let the user decide whether to iterate via opencode or fix in Claude

---

## Error Handling

- Stop and report failures when `opencode run` exits non-zero.
- Before dispatching builds, confirm model choice with user via `AskUserQuestion`.
- When output includes warnings or partial results, summarize and ask how to adjust.
- If model is unavailable (account limitation), fall back: try `opencode/nemotron-3-ultra-free` (always available, free).

---

## Session Management

```bash
# List all sessions
opencode session list

# Delete a session
opencode session delete <session-id>

# Export session as JSON
opencode export <session-id>

# View token usage
opencode stats
```

---

## Nemotron-3-Super Cookbook

Nemotron-3-Super-120B-A12B is a **120B total / 12B active-parameter** hybrid Mamba-Transformer MoE from NVIDIA. The free tier confirmed working through opencode is `opencode/nemotron-3-ultra-free` — `opencode models` only ever lists an "ultra" free variant, never a "super" one, so treat the specs below (NVIDIA's published Super architecture doc) as indicative, not verified against what "ultra" actually runs.

### Key Specs

| Spec | Value |
|------|-------|
| Architecture | Hybrid Mamba-2 + Transformer + Latent MoE |
| Total params | 120B (12B active per token) |
| Context window | **1M tokens** (native, Mamba-2 linear-time) |
| Code speedup | **2-3x** via multi-token prediction (MTP) — no draft model needed |
| Training data | 25T+ tokens (crawl + 15M coding problems + synthetic) |
| Benchmarks | Leads 120B-class on SWE-Bench Verified, Terminal Bench, AIME 2025 |
| Quantization | GGUF via Unsloth (Dynamic 2.0) — 12B active fits 16GB for inference |

### Optimal Prompting for Code Builds

- **Use high reasoning budget** for complex multi-file projects: `--variant high`
- **Temperature 1.0 / top_p 0.95** is the recommended config for code gen (per NVIDIA cookbook)
- **Agentic workflows** are its strength — multi-step tool-calling without goal drift
- The MTP heads specialize in **structured output** (code, tool calls, JSON) — 2-3x faster than standard autoregressive
- For Rust builds: the model handles code fluency across languages but has no Rust-specific fine-tuning — pair with thorough SPEC.md

### Reasoning Modes (API-level)

Nemotron supports `enable_thinking`, `reasoning_budget`, and `low_effort` modes:

| Mode | When to use |
|------|-------------|
| High budget (default) | Multi-file builds, complex architecture decisions |
| Low effort | Quick fixes, simple edits, formatting tasks |

In opencode, these map to `--variant high` / `--variant minimal`.

### Local GGUF on a 16GB laptop

The 12B active parameter count means quantized versions can run locally:
- Use Unsloth GGUF quants from HuggingFace: `unsloth/NVIDIA-Nemotron-3-Super-120B-A12B-GGUF`
- Q4_K_M fits in 16GB RAM for inference
- Set `baseURL: "http://localhost:8000/v1"` with `--special` flag for `<think>` tokens
- Default context: 262K (up to 1M if hardware supports it)

### vs GPT-5.x Codex for Builds

| | Nemotron-3-Super | GPT-5.x Codex |
|---|---|---|
| **Cost** | Free (opencode/nemotron-3-ultra-free) | Paid (API or ChatGPT sub) |
| **Context** | 1M tokens | Varies by model |
| **Speed** | 2-3x on code via MTP | Fast but no MTP advantage |
| **Integration gaps** | Same pattern — misses env plumbing, PID lifecycle | Same pattern |
| **Best for** | Free builds, long-context projects, agentic multi-step | Complex reasoning, latest training data |

**Recommendation:** Start with nemotron-3-ultra-free. Escalate to gpt-5.6-terra only if nemotron's output quality is insufficient for the task.

---

## OpenCode Environment & Advanced Config

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `OPENCODE_DISABLE_MODELS_FETCH=true` | Skip remote model list fetch |
| `OPENCODE_DISABLE_AUTOCOMPACT=true` | Disable auto context compaction |
| `OPENCODE_EXPERIMENTAL_FILEWATCHER=true` | Enable file watcher for dir monitoring |
| `OPENCODE_EXPERIMENTAL_LSP_TOOL=true` | Auto-configure language server (Rust analyzer, etc.) |

### Agents (Custom Workflows)

OpenCode supports custom agents with specialized prompts and tool permissions:

```bash
# List agents
opencode agent list

# Create a new agent
opencode agent create
```

Agents have separate permission sets (build, plan, explore, etc.) — see `opencode agent list` for the active ones. The default `build` agent has full write permissions.

### Plan Mode vs Build Mode

In the TUI, **Tab** switches between modes:
- **Plan mode**: Read-only outlining, architecture decisions — always plan first for large projects
- **Build mode**: File edits, code generation

When using `opencode run` (non-interactive), the model decides its own mode. For scaffolded builds, the SPEC.md serves as the plan.

---

## Notes

- `2>/dev/null` suppresses thinking tokens on stderr — always append unless debugging
- nemotron-3-ultra-free is the go-to for free builds (no API key needed)
- The spec is the contract — make it thorough so the model builds correctly
- Always include config files with real paths/URLs from the current machine
- After build: test against live systems, fix integration gaps, commit
- Run `opencode run` in background for long builds: add `run_in_background: true` to the Bash tool call
- Timeout: set 600000ms (10 min) for builds, models can be slow
- Sources: [NVIDIA cookbook](https://docs.nvidia.com/nemotron/nightly/usage-cookbook/Nemotron-3-Super/OpenScaffoldingResources/README.html), [Unsloth GGUF](https://huggingface.co/unsloth/NVIDIA-Nemotron-3-Super-120B-A12B-GGUF), [opencode docs](https://opencode.ai/docs/cli/)
