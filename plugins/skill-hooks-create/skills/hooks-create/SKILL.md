---
name: hooks-create
description: Create a hook for an AI CLI (claude-code, copilot, codex, opencode) or a git hook. Grills user/agent through the patterns that prevent silent data loss before any code gets written. Use when asked to "add a hook", "wire up a PreToolUse hook", "log every tool call", "block npm commands", "install a post-commit hook", or any similar request involving a CLI lifecycle hook or a git hook.
---

# /hooks-create — Hook Authoring Protocol

Use **before** writing any hook. Skipping the grill is how a post-commit hook ships with three portability bugs that silently corrupt its own output on every machine but the author's — across every repo it was installed in.

## Source-of-truth references

Hook APIs move. Before writing, check the current docs for the surface you picked —
the event names, payload shape, and blocking semantics below are a snapshot, not a contract:

- **Claude Code** — <https://docs.claude.com/en/docs/claude-code/hooks>
- **GitHub Copilot CLI**, **OpenAI Codex CLI**, **opencode** — each vendor's own hook reference
- **Git** — `man githooks`

If you can't reach them, the patterns below are a self-contained subset.

---

## The grill

Walk the user through these questions **before writing any hook code**. Don't accept "just write it" — every skipped question is a bug-shaped opening.

### 1. Which surface?

- [ ] **Claude Code** (`~/.claude/settings.json` or project `.claude/settings.json`)
- [ ] **GitHub Copilot CLI** (`.github/hooks/*.json`)
- [ ] **OpenAI Codex CLI** (`~/.codex/hooks.json` or `[hooks]` in `config.toml`)
- [ ] **opencode** (`.opencode/plugins/*.ts` — JS/TS, not shell)
- [ ] **Git hook** (`.git/hooks/<event>`)

Different surfaces = different config formats, different payload shapes, different escape hatches. Don't guess — confirm.

### 2. Which event?

| Surface | Common events |
|---|---|
| Claude Code | `SessionStart`, `UserPromptSubmit`, `PreToolUse`, `PostToolUse`, `Stop`, `SessionEnd`, `PreCompact` |
| Copilot CLI | `sessionStart`, `userPromptSubmitted`, `preToolUse`, `postToolUse`, `agentStop`, `errorOccurred` |
| Codex CLI | `PreToolUse`, `PostToolUse`, `UserPromptSubmit` (only `shell` tool fires reliably; `apply_patch` and most MCP bypass — verify) |
| opencode | `tool.execute.before/after`, `permission.asked/replied`, `session.created/idle/error`, `file.edited`, `shell.env` |
| Git | `post-commit`, `post-rewrite`, `pre-commit`, `commit-msg`, `pre-push`, `post-merge` |

### 3. What is the hook actually doing?

Pick one (or two — three is a smell):

- **Audit/log** — passive recording; never blocks; goes to a local file
- **Guardrail/block** — denies a tool call based on input
- **Transform args** — modifies the action before it runs
- **Training signal** — emits an event to a downstream stream or ledger
- **Workflow trigger** — runs formatter/test/build after edits

If you can't answer in one sentence, the hook is too big. Split it.

### 4. What's the failure mode?

- **Block** — exit 2 (CC) / `permissionDecision: deny` (CC, copilot) / throw (opencode) / return blocking JSON (codex)
- **Fail-open** — non-zero exit logged but agent continues (copilot is *always* fail-open; CC depends on event)
- **Fire-and-forget** — hook writes to a queue and returns immediately

Most audit hooks should be **fail-open**. Most guardrail hooks should be **blocking**. Mixing them up means a logging bug stops the agent.

### 5. Performance budget?

Hooks block the agent until they return. Hot-path events (PreToolUse, PostToolUse) should be **<100ms**. Slow work (network call, heavy parse) should be moved to async fire-and-forget pattern: hook writes a line to a queue file, separate runner processes it.

### 6. Where does output go?

- stdout JSON (CC, codex, copilot) → influences agent behavior
- stderr → user-visible logs
- local file (jsonl, append-only) → durable signal
- network → only if you can guarantee <100ms or async

---

## Anti-patterns (these caused real bugs — don't repeat them)

| ❌ Don't | ✅ Do | Why |
|---|---|---|
| `grep -oP "Merge branch '\K[^']*"` | `sed -n "s/^Merge branch '\\([^']*\\)'.*/\\1/p"` | `-P` is GNU-only; macOS BSD grep silently fails to the fallback |
| `git rev-list ...origin/main..` | Detect via `git symbolic-ref refs/remotes/origin/HEAD` with fallback ladder | Hardcoded `main` silently logs zero on master/develop repos |
| `cat << EOF { "branch": "$BRANCH" } EOF` | `jq -nc --arg branch "$BRANCH" '{branch: $branch}'` | Heredoc breaks on `"` / `\` in values; jq escapes correctly |
| `set -e` in post-commit | `set -uo pipefail` + explicit error handling | post-commit `set -e` produces confusing exit-code errors that don't undo the commit |
| `echo "logged"` (stdout) | `echo "logged" >&2` (stderr) | stdout is parsed by some hook callers; clutters scripted git pipelines |
| No timeout configured | `"timeout": 10` (CC) / `"timeoutSec": 10` (copilot) | Hung hook = hung agent |
| Trusting JSON input fields raw in shell | `jq -r '.field'` and **always quote** the result | Tool-input injection is real; quotes prevent shell expansion |
| No version stamp | `# === <NAME>_HOOK v=1.0 source=... ===` | Installers grep for the marker; version lets you migrate |
| No quiet mode | Honor `<NAME>_QUIET=1` env var | CI workflows need silence; tooling pipelines parse stdout |
| Hardcoded paths | `$CLAUDE_PROJECT_DIR`, or your own `$MYHOOK_DIR` override | Every machine has its own layout |
| Skipping fixture testing | Run hook against captured stdin **before** wiring into settings.json | Hooks are hard to debug live; fixtures aren't |

---

## Templates (minimal, opinionated)

### Claude Code — audit log on every tool call

`.claude/settings.json`:
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/audit.sh",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

`.claude/hooks/audit.sh`:
```bash
#!/usr/bin/env bash
# === MYHOOK_HOOK v=1.0 source=.claude/hooks/audit.sh ===
set -uo pipefail
LOG_DIR="${MYHOOK_LOG_DIR:-$CLAUDE_PROJECT_DIR/.claude/logs}"
mkdir -p "$LOG_DIR"
jq -nc --arg ts "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
       --slurpfile payload /dev/stdin \
       '{ts: $ts, payload: $payload[0]}' \
  >> "$LOG_DIR/audit-$(date -u +%Y-%m-%d).jsonl"
[ "${MYHOOK_QUIET:-0}" = "1" ] || echo "📝 audit logged" >&2
```

### Claude Code — block dangerous Bash

```bash
#!/usr/bin/env bash
# === BASHGUARD_HOOK v=1.0 ===
set -uo pipefail
input=$(cat)
cmd=$(echo "$input" | jq -r '.tool_input.command // empty')
if echo "$cmd" | grep -qE 'rm -rf /|:\(\)\{|sudo rm'; then
  jq -nc --arg reason "blocked dangerous command: $cmd" \
    '{hookSpecificOutput: {permissionDecision: "deny", reason: $reason}}'
  exit 0
fi
```

### Git — post-commit with version stamp + portable patterns

```bash
#!/usr/bin/env bash
# === MYHOOK_HOOK v=1.0 source=hooks/my-post-commit.sh ===
set -uo pipefail

# Detect default branch (NOT hardcoded)
DEFAULT_REF=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/@@')
[ -z "$DEFAULT_REF" ] && DEFAULT_REF="origin/main"

# Portable branch extraction (sed, NOT grep -oP)
COMMIT_MSG=$(git log -1 --pretty=%B)
MERGED=$(printf '%s\n' "$COMMIT_MSG" | sed -n "s/^Merge branch '\\([^']*\\)'.*/\\1/p" | head -n1)

# Build JSON via jq, NOT heredoc
jq -nc --arg branch "${MERGED:-unknown}" --arg default "$DEFAULT_REF" \
  '{branch: $branch, default_ref: $default}' \
  >> "$HOME/.local/state/myhook.jsonl"

[ "${MYHOOK_QUIET:-0}" = "1" ] || echo "logged: ${MERGED:-?}" >&2
```

### opencode — block reads of `.env`

`.opencode/plugins/env-guard.ts`:
```typescript
export const EnvGuard = async () => ({
  "tool.execute.before": async (input, output) => {
    if (input.tool === "read" && output.args.filePath?.includes(".env")) {
      throw new Error("🚫 Cannot read .env files");
    }
  },
});
```

---

## Test before you ship

A hook is hard to debug live and easy to debug as a fixture. Always:

1. **Capture a real payload once.** Add `tee /tmp/hook-fixture.json` to the top of the hook, fire it once via the agent, save the file.
2. **Run the hook against the fixture standalone:**
   ```bash
   cat /tmp/hook-fixture.json | bash my-hook.sh
   ```
3. **Write a regression test** — a bash harness that builds a fixture git repo in a temp dir, runs the hook against it, and asserts on the output. A hook with no test is a hook that fails silently on someone else's machine.
4. **Test the failure modes too** — what happens when stdin is empty? When the JSON is malformed? When `jq` is missing?

If you skip the test, expect to find your bug in production via the consortium reviewing your stream entries.

---

## When NOT to use a hook

- **Permanent policy** that should fire across all sessions and machines → may belong in the CLI's permissions config, not a hook
- **Workflow that needs to track state** across multiple events → consider an MCP server or a separate daemon, not a hook
- **Anything that needs to survive `claude /clear`** → hook config does, hook *state* doesn't
- **Cross-machine coordination** → hooks are local-only; use the Phase 2.2 stream-to-pool pipeline + Supabase for cross-machine signal

---

## Final checklist

Before committing the hook:

- [ ] Version-stamped header with `# === NAME_HOOK v=X.Y ===`
- [ ] No `grep -P`, no hardcoded default-branch refs, no heredoc-built JSON
- [ ] All shell vars quoted
- [ ] Stderr (not stdout) for human messages
- [ ] `<NAME>_QUIET=1` honored
- [ ] Explicit timeout in the registering settings file
- [ ] Fixture-based test passing
- [ ] If a guardrail: failure mode (block vs fail-open) is intentional
- [ ] If a training-signal: lands in `streams/*.jsonl` so Phase 2.2 can pick it up
- [ ] Cross-platform if the repo is shared (bash + powershell variants for Copilot CLI)
- [ ] Documented in the relevant CLI's hooks doc (or in `AGENT_CLI_HOOKS_KB.md`)

---

*"A hook is a yes-or-no question the CLI asks before it acts. Answer in milliseconds, in writing, with your reasoning logged."*
