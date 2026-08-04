---
name: codex
description: Use when the user asks to run Codex CLI (codex exec, codex resume) or references OpenAI Codex for code analysis, refactoring, or automated editing
---

# Codex Skill Guide

## Pre-flight: Check Login

Before running any `codex exec` command, verify login status:

```bash
codex login status 2>&1
```

**If not logged in**, recover using one of:

```bash
# Option 1 — Device auth (preferred; no secret ever touches argv)
codex login --device-auth

# Option 2 — Interactive ChatGPT login (requires a real terminal)
# Tell the user to run it themselves: ! codex login

# Option 3 — API key, headless. Feed it on STDIN, never as an argument.
printenv OPENAI_API_KEY | codex login --with-api-key
```

If login fails mid-run (non-zero exit plus an auth error in the output), run the
recovery above once, then retry. Do **not** loop — if it fails twice, stop and ask
the user.

**Never put a secret where a shell can expand it into `argv`.** A wrapper of the
shape `some-tool --key "$(cat ~/.secrets/key)"` expands *before* exec, so the key's
literal contents become a command-line argument — readable by any process the same
user can run (`ps -eo args`), and often captured in shell history and process
accounting. Masking options on such tools typically cover the child's stdout/stderr
and do not touch argv. Pass secrets on stdin (as Option 3 does) or via an
environment variable the tool reads itself.

**Do not silently switch auth method.** Falling back from a ChatGPT session to an
API key can move usage onto API billing and change which account you are acting as
— and the two sign-in modes do not expose the same model list, so a slug that
"disappeared" may just be one the current mode never offered. Ask before changing it.

> **Note on model availability:** ChatGPT sign-in and API-key sign-in do not expose the
> same model list, and the deprecations in **Models** below are specifically scoped to
> ChatGPT sign-in. If a model slug is rejected, check which auth mode is active before
> assuming the model is gone.

## Models

**Current as of 2026-08-02.** Codex ships model families faster than this file gets
updated, so treat the table as a floor, not a ceiling. The failure mode to avoid is
falling back to pre-GPT-5.6 names (`gpt-5.4`, `gpt-5.3-codex`, `gpt-5.2`) just because
they are the ones you remember from training — those are deprecated or already retiring.
If the installed CLI offers something newer than what is listed here, prefer the newer
thing and update this table.

### The GPT-5.6 family (GA in Codex 2026-07-09) — the default choice

| Slug | Tier | Reach for it when |
|------|------|-------------------|
| `gpt-5.6-sol` | Deep-reasoning flagship | Ambiguous, high-value, architecturally hard work — complex refactors, deep debugging, design review |
| `gpt-5.6-terra` | Everyday workhorse | **Default for most dispatches.** Roughly GPT-5.5-class at about half the price |
| `gpt-5.6-luna` | Fast / cheap | High-volume, latency-sensitive, mechanical work — bulk edits, subagents |

### Also selectable

| Slug | Status |
|------|--------|
| `gpt-5.5` | Previous frontier default (since 2026-04-23). Superseded by `gpt-5.6-terra`/`-sol` for most work |
| `codex-auto-review` | Purpose-built reviewer/guardian model, `visibility: hide` in the catalog. **Not** a general driver — do not dispatch ordinary tasks to it |

`gpt-5.3-codex-spark` (sub-second latency, 128K text-only) is **absent** from the current
`models_cache.json` and must not be offered as a choice. Historical note only.

### Deprecated / scheduled retirement — do not select for new work

| Slug | What happened | Replace with |
|------|---------------|--------------|
| `gpt-5.4` | Retires from Codex with ChatGPT sign-in on **2026-08-31** | `gpt-5.6-terra` |
| `gpt-5.4-mini` | Retires from Codex with ChatGPT sign-in on **2026-08-31** | `gpt-5.6-luna` |
| `gpt-5.3-codex` | Deprecated for ChatGPT sign-in | `gpt-5.6-terra` |
| `gpt-5.2` | Deprecated for ChatGPT sign-in | `gpt-5.6-sol` |

⚠️ **The 2026-08-31 retirement is imminent.** If `~/.codex/config.toml`, a custom agent,
or a scheduled task still names `gpt-5.4` or `gpt-5.4-mini`, say so and offer to migrate
it before dispatching.

### Reasoning effort

Passed as `--config model_reasoning_effort="<effort>"`. **Effort support and the default
are per-model, not global.** Verified against `~/.codex/models_cache.json` on 2026-08-03:

| Model | Default | Supported efforts |
|-------|---------|-------------------|
| `gpt-5.6-sol` | **`low`** | low, medium, high, xhigh, max, **ultra** |
| `gpt-5.6-terra` | `medium` | low, medium, high, xhigh, max, **ultra** |
| `gpt-5.6-luna` | `medium` | low, medium, high, xhigh, max |
| `codex-auto-review` | `medium` | low, medium, high, xhigh, max |
| `gpt-5.5` | `medium` | low, medium, high, xhigh |
| `gpt-5.4` | `medium` | low, medium, high, xhigh |
| `gpt-5.4-mini` | `medium` | low, medium, high, xhigh |

**`gpt-5.6-sol` defaults to `low`.** Dispatching the deep-reasoning flagship without an
explicit effort gets you its *weakest* setting — and `--ignore-user-config` drops the
config default too, so the trap fires precisely in headless dispatches. Always pass
effort explicitly with `sol`.

`ultra` is a real effort value, on `sol` and `terra` only — maximum reasoning with
automatic delegation to parallel subagents. It is not a separate feature reached some
other way. `max` is real on every GPT-5.6 model and on `codex-auto-review`; the pre-5.6
models cap at `xhigh`.

`minimal` and `none` are recognized by the CLI's own enum but are **advertised by no
cached model**, so a request for either is normalized or dropped rather than honoured.
The per-model list above is authoritative over the CLI's accepted-value set.

An unsupported value is **coerced rather than rejected** — a dispatch behaving like `low`
when you asked for more may mean the value never took, not that the model underperformed.

### Choosing, and refreshing this table

Do not guess from memory — it goes stale in weeks. Check, in order:

1. `~/.codex/config.toml` — the `model =` line is the user's chosen default.
2. `~/.codex/models_cache.json` — what this install last saw the API offer.
3. The `/model` picker in an interactive `codex` session.
4. <https://developers.openai.com/codex/models> — authoritative.

**Prefer the user's configured default**, but if it names a retired model, flag it instead
of silently using it. Ask for model + reasoning effort together in one `AskUserQuestion`
when the user has not specified them.

## Running a Task

1. Check login (Pre-flight above).
2. Ask model + reasoning effort in one `AskUserQuestion` if not given. Offer current slugs from **Models** (`gpt-5.6-terra` default) — never a pre-5.6 name.
3. Sandbox: `read-only` unless the task genuinely writes. Note what `read-only` does and
   does not mean — it constrains **writes**, not command execution. A `read-only` dispatch
   still runs arbitrary shell (`/bin/zsh -lc ...`), so it can read any file the user can
   read and reach the network. It is a write guard, not a containment boundary. Never pick
   `danger-full-access` merely because a task wants network.
4. Assemble the command. **`-a never` is TOP-LEVEL, not `exec`-level — it goes BEFORE `exec`:**
   ```
   codex -a never exec [options] "prompt"
   ```
   `codex exec -a never ...` errors with `unexpected argument '-a' found`.

   | Flag | |
   |---|---|
   | `-a never` | approval policy — **before** `exec` |
   | `-m, --model <MODEL>` | |
   | `--config model_reasoning_effort="<effort>"` | |
   | `--sandbox <read-only\|workspace-write\|danger-full-access>` | |
   | `-C, --cd <DIR>` | run from another directory |
   | `--skip-git-repo-check` | **only** when the target really is outside a git repo |
   | `"prompt"` | final positional argument |

   **NEVER pass `--full-auto`. It silently escalates your sandbox.** It is a real, hidden,
   deprecated `exec`-level flag — not the no-op earlier versions of this skill claimed. It
   forces `workspace-write` and approval `never`, **overriding an explicit
   `--sandbox read-only` that appears later on the same command line**.

   Verified on Codex 0.146.0, 2026-08-03:

   | Command | approval | sandbox |
   |---|---|---|
   | `codex exec --sandbox read-only` | `on-request` | `read-only` |
   | `codex exec --full-auto --sandbox read-only` | `never` | **`workspace-write [workdir, /tmp, $TMPDIR]`** |

   The only signal is `warning: --full-auto is deprecated` on **stderr** — which the
   `2>/dev/null` in older examples would discard. Believing the flag is inert while
   discarding the one message that says otherwise is how an agent ends up with write
   access to the workdir and `/tmp` while thinking it has read-only. Non-interactive means
   `-a never` plus an explicit `--sandbox <mode>`, and nothing else.

   **`--skip-git-repo-check` is not a default either.** It disables the guard that makes
   Codex refuse an untrusted directory, so a malformed `-C` proceeds in `$HOME` or a parent
   directory instead of failing closed. Inside a repo, omit it. Outside one, verify the
   absolute path first, and ask before pairing it with a writable sandbox.
5. **`-a never` on every non-interactive dispatch.** Without it codex blocks on a TTY-only "do you trust this folder?" prompt the first time it runs in a directory — invisible and unanswerable behind `2>/dev/null | tail -N`. It does *not* bypass sandboxing (that is `--dangerously-bypass-approvals-and-sandbox`, unused here); a would-be approval returns to the model as a failure instead.
6. **`--ignore-user-config` by default for headless code-fix dispatches** (`exec`-level). Skips `~/.codex/config.toml`, so no MCP servers initialize — see hang mode 2, which `-a never` does *not* cover. Auth still works via `CODEX_HOME`. It also drops the config's model + effort, so pass both explicitly. Keep the flag off only when a task genuinely needs an MCP server.
7. **Do not blanket-discard stderr.** `2>/dev/null` hides thinking tokens, but it also
   hides deprecation warnings, auth failures, the resolved sandbox, and which model was
   actually selected — the exact evidence Error Handling below tells you to inspect.
   Redirect to a file instead and read it on any non-zero exit or surprising result:
   `2>/tmp/codex-<task>.err`. Keep `2>/dev/null` only for runs whose stderr you have
   already decided you will never need.
8. Run, summarize, then tell the user they can resume any time by saying "codex resume".

### Quick Reference

Base invocation — note it carries `--ignore-user-config` and an explicit effort, as
steps 6 and 7 require, and sends stderr to a file rather than discarding it:

```bash
codex -a never exec --ignore-user-config \
  -m gpt-5.6-terra --config model_reasoning_effort="high" \
  --sandbox read-only "prompt" 2>/tmp/codex-review.err
```

| Use case | Change from the base |
|---|---|
| Read-only review | base as written |
| Apply local edits | `--sandbox workspace-write` |
| Run from another dir | add `-C <ABSOLUTE_DIR>` |
| Target is outside a git repo | add `--skip-git-repo-check`, after verifying the absolute path |
| Resume a session | `resume <SESSION_ID>` with the prompt on stdin — see **Following Up** |
| Network access | **Do not reach for `--sandbox danger-full-access`.** It grants unrestricted filesystem *and* network to model-generated commands. Needing to fetch a doc is not a reason to disable the sandbox — narrow the task, fetch the content yourself and pass it in, or tell the user this environment cannot grant network safely. |

## Following Up

- `AskUserQuestion` after every dispatch — next steps, or whether to resume.
- **Never resume with `--last`.** It resolves to whichever Codex session finished most
  recently, machine-wide. Another agent's dispatch completing in between silently hijacks
  your resume, and under a writable sandbox that edits the wrong project. Capture the
  session id from the dispatch banner (`session id: <uuid>`, or use `--json`) and resume
  that id explicitly.
- **Resume does not reliably inherit model, effort, and sandbox.** The resume path
  re-resolves current config and passes its own model and sandbox, so a config change
  between dispatch and resume silently alters them. Pass `-m`, the effort, and
  `--sandbox` explicitly whenever continuity matters.
- Restate model, effort, and sandbox when proposing follow-ups.

## Critical Evaluation of Codex Output

Codex has its own knowledge cutoff and failure modes. **Colleague, not authority.**

- Push back on claims you know are wrong. Research disagreements (WebSearch, docs) before accepting them.
- **Recency cuts both ways.** Codex may not know recent releases — and neither may you. Check the lockfile or the web before asserting in either direction.
- Do not grade on effort. Long, confident, and wrong is still wrong.

When Codex is wrong: state the disagreement, give evidence, optionally resume peer-to-peer identifying yourself as Claude, and let the user decide genuine ambiguity.

```bash
echo "This is Claude (<model>) following up. I disagree with [X] because [evidence]. Your take?" \
  | codex -a never exec --skip-git-repo-check resume --last 2>/dev/null
```

For anything expensive to get wrong, do not freehand it — use **`/clodex`**: two reviewers aimed at different failure classes, every claim verified before it is relayed, and the fixes split under a contract negotiated rather than assigned.

## Contract-Driven Dispatch (STATE.json)

For projects carrying `docs/arch/STATE.json` + `docs/arch/contracts/`.

Lifecycle: `pending → specced → codex-dispatched → review → done`

1. Read `STATE.json`, find the next `specced` candidate.
2. Read its contract file.
3. Pipe the contract in as the prompt:
   ```bash
   contract=$(cd "<project-dir>" && pwd)/docs/arch/contracts/<n>-<name>.md
   codex -a never exec --ignore-user-config -m <model> \
     --config model_reasoning_effort="high" \
     --sandbox workspace-write -C "<project-dir>" - \
     < "$contract" 2>/tmp/codex-contract.err
   ```

   **Pass the contract on stdin, with an absolute path.** The obvious
   `"$(cat docs/arch/contracts/...)"` form is broken twice over: the shell expands `$(cat …)`
   in *your* cwd before Codex ever applies `-C`, so it reads the wrong project's contract or
   fails outright, and a large contract can blow the argv limit. Despite the surrounding
   prose calling it a pipe, that form is not one.
4. Set `status: "codex-dispatched"` and `codex_session: "last"`.
5. On completion → `status: "review"`; verify against the contract's **Success criteria**.
6. After human sign-off → `status: "done"`.

Every contract file needs all of: **Goal**, **Seam** (types, methods, invariants), **Files to touch**, **Files NOT to touch**, **Tests required**, **Success criteria** (checkable), **Out of scope**, and — critically — **do not remove existing tests**: add only. Codex rewrites test files and silently drops existing cases unless told otherwise. Do not dispatch until every section is filled.

## Error Handling

Stop and report on any non-zero exit; ask before retrying. Auth errors → pre-flight recovery, retry once. Ask via `AskUserQuestion` before `--sandbox danger-full-access` or `--skip-git-repo-check` unless already granted. Summarize warnings and partial results, then ask how to adjust.

### Hang mode 1 — "do you trust this folder?"

0% CPU, blocked on invisible TTY input, first run in a new directory. Re-dispatch with
`-a never`. Confirmed 2026-05-13: 27 min at 0% CPU before the kill.

**Kill by recorded PID, never by pattern.** `pgrep -f "codex exec"` does not kill anything
on its own, and its pattern is not session-specific: it matches every concurrent Codex on
the machine — including other agents' `workspace-write` dispatches, which a pattern-kill
would interrupt mid-edit. It also matches *your own* wrapper shell if that shell's command
line happens to contain the string, which turns an `until ! pgrep …` wait into an infinite
loop. Capture `$!` when you launch and kill only that PID.

### Hang mode 2 — MCP-server init (survives `-a never`)

`codex exec` initializes every MCP server declared in `~/.codex/config.toml` *before* doing any work; one that fails to come up blocks forever.

- **0% CPU is not by itself the signature.** A healthy dispatch sits at 0.0% CPU for
  minutes at a time while blocked on the model API. Judge by *progress*, not CPU: a live
  run has a `rollout-*.jsonl` and a growing stderr file. Killing on CPU alone will
  routinely kill working dispatches.
- **Diagnostic:** `~/.codex/sessions/<YYYY>/<MM>/<DD>/` — a working run writes `rollout-*.jsonl` almost immediately. Nothing recorded means it never got past startup, not that it is running slowly. Confirm servers are configured: `grep -c '\[mcp' ~/.codex/config.toml`.
- **Fix:** `--ignore-user-config`. Skips config.toml entirely; auth still works via `CODEX_HOME`. Pass `-m` and `--config model_reasoning_effort` explicitly, since the config's defaults go with it.
- Confirmed 2026-05-19: three dispatches hung 98 min at 0% CPU *with* `-a never` set, recording zero session files; a probe with `--ignore-user-config` exited 0 in under 100s.

### Hang mode 3 — writes flush after success

A `--sandbox workspace-write` dispatch can exit 0 and fire the caller's task notification while still flushing files for a few seconds. New changes can appear after you already reviewed, tested, and committed.

**Compare content hashes, not `git status` text.** Two `git status` runs are *not* a
stability check. A file already listed as ` M src/auth.ts` that a late writer changes
again is still ` M src/auth.ts` — identical status output, different bytes. Verified
2026-08-03: same porcelain output across a modification, different `shasum`.

Do this instead:

```bash
wait "$codex_pid"                        # exited process cannot flush more writes
snap() { git status --porcelain | awk '{print $2}' | xargs shasum 2>/dev/null; }
a=$(snap); sleep 5; b=$(snap)
[ "$a" = "$b" ] && echo "tree stable"
```

Waiting for the dispatch process and its descendants to exit is the real guarantee; the
hash re-check is the backstop for anything it spawned and orphaned.

**Never discard late changes just because they look redundant.** Resemblance is not
provenance: those bytes may be the user's own edit, or another agent's, arriving in a
shared checkout. Establish where they came from, keep a patch or backup, and get approval
before discarding anything.
