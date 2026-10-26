---
name: memory-trim
description: >
  Measure and trim what Claude Code loads into context at startup — the auto-memory
  index (MEMORY.md), memory files, and the CLAUDE.md chain — then prove the saving with
  before/after /context numbers. Use when a session starts heavier than it should, when
  two machines start at different context sizes, or for periodic memory hygiene
  ("memory pass", "trim memory", "why is my context bigger on this machine").
---

# /memory-trim — Measure, then trim, startup memory

Startup context is assembled from local files that **do not sync between machines**:
the auto-memory dir under `~/.claude/projects/<project-slug>/memory/`, every `CLAUDE.md`
from `~` down to the working directory, installed skills, plugins and MCP servers. Each
host drifts on its own. This skill measures first, trims only what is provably stale,
and measures again.

## When to Use

- One machine starts noticeably heavier than another on the same project.
- `MEMORY.md` has grown prose instead of one line per memory.
- Memories mention tasks, flags, or systems you suspect are gone.

Not for: editing a repo's committed `CLAUDE.md` (that is a doc review with its own PR),
or touching another machine's files.

## Process

### 1. Measure (baseline)

`/context` works headless, so the breakdown can go to a file. It can hang, and macOS has
no `timeout`, so bound it with `perl`'s alarm:

```bash
cd <project-dir> && perl -e 'alarm 240; exec @ARGV' claude -p "/context" < /dev/null \
  > ~/context-$(hostname -s)-before.txt
grep -E 'Tokens:|Memory files|Skills|MCP' ~/context-$(hostname -s)-before.txt
```

Run it **from the same directory** on every machine you compare — the `CLAUDE.md` chain
depends on the cwd. Reading the output:

- `Autocompact buffer` is reserved, identical everywhere; ignore it.
- `MCP tools (deferred)` / `System tools (deferred)` are **not** in the token total —
  only their names load. A big deferred number is not your gap.
- The `Memory Files` table lists every `CLAUDE.md` and `MEMORY.md` with its tokens.
  That table is usually where the difference lives.
- `/context` shows only **this** project's `MEMORY.md`. For other memory dirs, measure
  bytes (step 2) before and after.

**Comparing two machines:** copy one file across and diff the sections:

```bash
diff <(sed -n '/### Memory Files/,/### Skills/p' a.txt) <(sed -n '/### Memory Files/,/### Skills/p' b.txt)
```

- **A repo's `CLAUDE.md` differs in size:** one checkout is at another commit. Look
  before pulling — the checkout may be on someone else's branch:
  `git fetch && git diff HEAD origin/main -- CLAUDE.md`.
- **A `CLAUDE.md` exists on one host only** (often `~/CLAUDE.md` or a parent dir's): a
  decision for the owner — keep on both or neither. See step 8.

### 2. Rank the memory dirs

Project slugs start with `-` (`-Users-me-project`), so always prefix globs with `./`
or pass `--`, or tools read them as options.

```bash
cd ~/.claude/projects
for d in ./*/memory; do [ -f "$d/MEMORY.md" ] && echo "$(wc -c < "$d/MEMORY.md") $d"; done | sort -rn | head
```

Roughly 4 bytes per token. Only `MEMORY.md` loads at startup; the other memory files
load on recall. Work the biggest index first; skip dirs under ~2k tokens.

Each memory dir is independent. For a big pass, run one agent per dir, and give each
the list of tools and systems your setup has retired — most stale memories trace back
to one of those.

### 3. Back up before touching anything

One directory per project and per run, so two trims on the same day never overwrite
each other. Stop if the copy fails — every later step edits or deletes files.

```bash
MEM="<memory-dir>"
BACKUP="$HOME/.claude/memory-backup/$(basename -- "$(dirname -- "$MEM")")-$(date +%Y%m%dT%H%M%S)"
mkdir -p -- "$BACKUP" && cp -p -- "$MEM"/*.md "$BACKUP"/ && echo "backup: $BACKUP" || echo "BACKUP FAILED — stop"
```

### 4. Read every memory, then classify each one

Read every file, not just the index and the ones that look stale.

| Verdict | When |
|---|---|
| **keep** | Still true, not recorded anywhere else |
| **condense** | True, but carries history or steps that no longer matter |
| **merge** | Two memories describe one fact or one resolved incident |
| **delete** | Stale, wrong, or already recorded in the repo (CLAUDE.md, git history, docs) |

**Stale is a claim — verify it against live state before deleting.** One command per
memory: does the named file, flag, command, table or config key still exist? Is the
cited task/issue still open? Is the "current mode" still current? Record what you
checked; "looks old" is not evidence. A memory that cites a now-closed issue is often
best **merged** into one short "resolved: what is true now" memory, not dropped — the
resolution may carry a residual caveat worth keeping.

**Feedback memories** (corrections the user gave): read them fully and prefer
condensing. Delete one only when the user retracted it, the tool it works around is
retired, or the repo's `CLAUDE.md` already states the rule — and name each deleted
feedback memory in the report.

### 5. Rewrite the index

`MEMORY.md` is an index: one line per memory, `- [Title](file.md) — hook`, grouped
under a few headings. No inline prose, tables, status snapshots or milestones — those
either become their own memory file or go.

**Line budget: hook ≤ ~80 characters** — when it applies, plus the rule. Short lines
usually save more than deletions do. A line already truncated with "…" still costs its
full length.

### 6. Check the links, both ways

```bash
cd -- "<memory-dir>"
# every index entry points at a file that exists
grep -oE '\]\([^)]+\.md\)' MEMORY.md | sed 's/^](//; s/)$//' | while read -r f; do [ -f "$f" ] || echo "missing: $f"; done
# every memory file is listed in the index
for f in ./*.md; do f=${f#./}; [ "$f" = MEMORY.md ] || grep -qF "($f)" MEMORY.md || echo "unlisted: $f"; done
# [[links]] to deleted memories
grep -rn "<deleted-name>" .
```

Repoint `[[name]]` links at the merged memory, or drop them.

### 7. Measure again and report

```bash
perl -e 'alarm 240; exec @ARGV' claude -p "/context" < /dev/null | grep -E 'Tokens:|Memory files|MEMORY.md'
```

Report: baseline before → after, `Memory files` before → after, `MEMORY.md` bytes
before → after for every other dir you trimmed, each deleted/merged memory with the
one-line reason and what you checked, and the backup path.

### 8. Removing or archiving a CLAUDE.md (owner decision)

- List what it holds that loads nowhere else, and grep the other `CLAUDE.md` files for
  pointers to it ("the full rules live in …"). Move those sections before archiving —
  a file that looks like boilerplate can carry the only copy of a rule.
- `~/CLAUDE.md` loads in **every** session under your home dir; a parent dir's
  `CLAUDE.md` loads only below that dir. Moving a rule between them changes where it
  applies.
- If a hook blocks agents from moving `CLAUDE.md` files, the owner runs the move.

## Rules

- Measure before and after; a pass with no numbers is not done.
- Never delete on age alone — verify first.
- Never edit another machine's memory; file a task for that machine instead.
- Keep the backup until the next session confirms nothing important went missing.
