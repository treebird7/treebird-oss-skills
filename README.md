# Treebird Public Skills

Public [Claude Code](https://claude.com/claude-code) skills from the Treebird flock.

| Plugin | Command | What it does |
|---|---|---|
| `skill-codex` | `/codex` | Delegate a task to the Codex CLI — model selection, sandbox modes, contract-driven dispatch, and the hang modes that bite in headless runs |
| `skill-clodex` | `/clodex` | Two-model adversarial review under a negotiated contract — each verifies the other's claims, then the fixes split by disjoint file set and land in one build |
| `skill-rls-audit` | `/rls-audit` | Audit a deployed Supabase project for multi-tenant RLS leaks — and *demonstrate* isolation with a two-tenant black-box matrix rather than inspecting it |

The first two are for working with OpenAI Codex from inside Claude Code — one to *drive*
it, one to *disagree* with it. The third is unrelated to Codex and has no dependency on it.

## Requirements

- Claude Code.
- **For `/codex` and `/clodex` only:** the `codex` CLI on your `PATH` (verify with
  `codex --version` and resolve any errors before installing — both skills shell out to it)
  and a working Codex login (ChatGPT sign-in, device auth, or an API key).
- **For `/rls-audit` only:** read access to the Supabase project you're auditing — the
  Supabase CLI linked to it (`supabase link`), the Supabase MCP server, or the SQL editor.
  `psql`, `curl`, and `jq` for the two-tenant matrix.

## Installation

### Option 1 — Plugin marketplace (recommended)

```
/plugin marketplace add treebird7/treebird-oss-skills
/plugin install skill-codex@treebird-oss-skills
/plugin install skill-clodex@treebird-oss-skills
/plugin install skill-rls-audit@treebird-oss-skills
```

`/clodex` calls into `/codex` for pre-flight login, error handling, and the Codex CLI
flag/model mechanics, so `skill-clodex` declares `skill-codex` as a plugin dependency.
Installing `skill-clodex` pulls `skill-codex` in with it — the line above is there so
you can install `/codex` on its own, not because you must install both by hand.

### Option 2 — Standalone skills

```bash
git clone --depth 1 https://github.com/treebird7/treebird-oss-skills.git /tmp/treebird-oss-skills && \
mkdir -p ~/.claude/skills && \
cp -r /tmp/treebird-oss-skills/plugins/skill-codex/skills/codex         ~/.claude/skills/codex && \
cp -r /tmp/treebird-oss-skills/plugins/skill-clodex/skills/clodex       ~/.claude/skills/clodex && \
cp -r /tmp/treebird-oss-skills/plugins/skill-rls-audit/skills/rls-audit ~/.claude/skills/rls-audit && \
rm -rf /tmp/treebird-oss-skills
```

`/rls-audit` ships a companion file (`tenant-matrix.md`) alongside its `SKILL.md`; copy the
whole directory, not just the `SKILL.md`.

## `/codex` — delegate to Codex

Ask for it in plain language:

> Use codex to review this module for concurrency bugs.

Claude will check your Codex login, ask which model and reasoning effort to use, pick a
sandbox mode (`read-only` by default), dispatch, and summarize the result.

What the skill actually buys you over typing `codex exec` yourself:

- **A current model table.** GPT-5.6 `sol` / `terra` / `luna`, what each tier is for, and
  the models that are already deprecated or retiring — so the agent stops reaching for
  `gpt-5.4` out of habit. See [Models and freshness](#models-and-freshness).
- **Flag positioning that actually works.** `-a never` is a *top-level* flag; `codex exec
  -a never` errors out. `--full-auto` is not a real flag at all.
- **Three documented hang modes**, each with a diagnostic and a fix: the invisible
  "do you trust this folder?" TTY prompt, MCP-server startup deadlock, and the
  asynchronous file flush that can land writes *after* a `workspace-write` dispatch has
  already reported success.
- **Contract-driven dispatch** — a `STATE.json` + contract-file pattern for handing Codex
  a scoped, bounded unit of work with explicit "files not to touch" and "do not delete
  existing tests" boundaries.

### Thinking tokens

Dispatches append `2>/dev/null` to suppress Codex's reasoning stream, which otherwise
floods Claude Code's context. Ask explicitly if you want to see it for debugging.

## `/clodex` — two reviewers, one contract

> Review this migration with clodex before we ship it.

`/clodex` exists because a single reviewer — and a second reviewer using the same
method — tends to agree with the first. The rule everything else serves: **the reviewer
that questions your premises beats the reviewer that checks your logic.**

1. **Review it yourself first**, at the source. Decompile the jar, read the pinned tag,
   mutate the constant and confirm the test goes red. You need findings to trade.
2. **Dispatch the second reviewer aimed away from your method** — numbered hunting
   grounds biased toward premises, environment, and lifecycle. Demand `file:line`, a
   concrete failure scenario, a **non-findings** section, and a falsifier.
3. **Verify every claim before relaying one.** Confirmed / Discounted / Unproven. Its
   findings are claims, not results.
4. **Negotiate the contract — don't dispatch it.** Disjoint file sets, named scope per
   finding, add-only tests, exactly one build. Demand objections, not agreement.
5. **Land both halves in one build**, mutation-checking the other party's tests too.

Design constraints worth knowing before you use it:

- **Capitulation is not resolution.** A reviewer that folds without engaging the
  evidence leaves the claim *Unproven*, not settled — and the rule binds both sides.
- **Zero findings is a signal, not a pass.** It suggests you reviewed the summary
  rather than the code.
- **Neither model is the tiebreaker.** Past two exchanges, the disagreement goes to a
  human with both cases stated at their strongest.
- **The second model is not smarter. It is differently blind.** The whole method rests
  on that.
- **It is not cheap** — roughly three dispatches plus your own high-effort pass. Don't
  point it at a rename; use it where being wrong is expensive and green CI proves
  little.

## `/rls-audit` — prove your tenants are actually isolated

> Audit this project for RLS leaks before we onboard the second customer.

Eight checks against a **deployed** Supabase project. Not a migration reviewer — it reads
the running system, on the assumption that the migration tree lies, because grants get
hand-edited in the SQL editor, tightening migrations silently no-op, and the table added
last month never got a policy.

The premise: **a single-tenant app cannot leak to itself.** Every bug this finds is dormant
until customer #2 signs up, which is exactly when it stops being cheap to fix.

What it covers that a policy review doesn't:

- **Views** — a view runs as its *owner* unless `security_invoker = true`, so a view over
  an RLS table returns every tenant's rows and the base table's policies are never
  consulted. Materialized views can't be fixed this way at all.
- **Claim provenance** — `user_metadata` in a Supabase JWT is **user-writable**. A policy
  reading the tenant id from there lets any user mint themselves membership of any tenant,
  and it reviews clean because it looks exactly like the correct pattern. The skill traces
  every tenant predicate back to its source and classifies it trusted or not.
- **Write policies** — a `WITH CHECK`-less `INSERT`/`UPDATE` policy means read is scoped and
  write is not: you can insert rows into another tenant, or move your own row into theirs.
  Read-only test suites never catch it.
- **Permissive-OR** — a narrow policy does not constrain a broad one beside it, so a
  `DROP POLICY IF EXISTS` with a stale name is a silent no-op that leaves the leak live.
- **The surfaces nobody audits** — `SECURITY DEFINER` RPCs (anon-callable by default),
  column-level grant breadth, public storage buckets and unpinned object paths, the
  `supabase_realtime` publication, and service-role keys in client bundles.
- **Existence oracles** — `count=exact` returning a positive number with an empty body, and
  unique-constraint violations whose error text confirms another tenant's row exists.

And the part that separates it from every checklist: **Check 7, the two-tenant matrix.**
Everything else *inspects*; this *demonstrates*. Two real tenants, twelve cases, anon key only
— never the service role, which carries `BYPASSRLS` and makes every case pass while proving
nothing. If the run didn't happen, the report has to say `ISOLATION DEMONSTRATED: no
(static only)` instead of implying the project is safe.

Design constraints worth knowing:

- **A clean advisor run is not a pass.** Supabase's linter checks structure — is RLS on, is
  the view a definer view. A table with RLS enabled and a policy of `USING (true)` is
  advisor-clean and wide open. The skill runs the advisors first and then keeps going.
- **Read-only by discipline.** Never `SELECT` real tenant rows to prove a finding — policy
  plus grant plus a reachable path is proof, and pasting another tenant's data into a
  transcript *is* the breach you're reporting.
- **Bound the blast radius.** Findings are reported with a verified-clean perimeter, because
  a finding without one turns into a week of undirected panic.
- **Check 8 exists because audits decay.** It proposes a CI gate — five catalog queries that
  must each return zero rows — so the table added next month can't quietly reintroduce
  what you just fixed.

## Models and freshness

The two Codex skills carry a dated model table rather than relying on memory, because model
knowledge goes stale in weeks and a stale table is worse than none — the agent
confidently reaches for a model that no longer exists.

As of **2026-08-02**:

| Slug | Tier | Notes |
|---|---|---|
| `gpt-5.6-sol` | Deep reasoning | Hard, ambiguous, high-value work |
| `gpt-5.6-terra` | Workhorse | Default pick for most dispatches |
| `gpt-5.6-luna` | Fast / cheap | High-volume, latency-sensitive work |
| `gpt-5.5` | Previous frontier | Still selectable |
| `gpt-5.3-codex-spark` | Research preview | Sub-second latency, 128K text-only |
| `codex-auto-review` | Reviewer/guardian | Purpose-built; not a general driver |
| `gpt-5.4`, `gpt-5.4-mini` | ⚠️ Retiring | Removed from Codex with ChatGPT sign-in on **2026-08-31** → use `terra` / `luna` |
| `gpt-5.3-codex`, `gpt-5.2` | ⚠️ Deprecated | For ChatGPT sign-in |

The skills instruct the agent to treat that table as a floor rather than a ceiling, and
to re-check `~/.codex/config.toml`, `~/.codex/models_cache.json`, the interactive
`/model` picker, and <https://developers.openai.com/codex/models> before choosing.

If you're reading this well after August 2026, the table is probably stale — that is
expected, and the skills say so out loud. PRs refreshing it are welcome.

## Repository layout

```
.claude-plugin/marketplace.json   # marketplace manifest — lists every plugin
plugins/
  skill-<name>/
    .claude-plugin/plugin.json    # plugin manifest — version must match the marketplace entry
    skills/<name>/SKILL.md        # the skill itself
```

### Adding a skill

1. `mkdir -p plugins/skill-<name>/skills/<name>` and write `SKILL.md` with YAML
   frontmatter carrying at least a `description:` — that field is what the skill loader
   reads to decide when to trigger, so make it say *when to use this*, not just what it is.
2. Add `plugins/skill-<name>/.claude-plugin/plugin.json`.
3. Add a matching entry to `.claude-plugin/marketplace.json`. **The `version` in both files
   must match** — they are read independently and a mismatch installs the wrong one.
4. Keep skills self-contained. A skill that shells out to a tool most people don't have
   belongs somewhere else, or needs its prerequisites stated up front.

Do not add a bare `skills/` line to `.gitignore` — see the note in that file.

## Contributing

Issues and PRs welcome, particularly:

- Model table refreshes as OpenAI ships new families.
- New Codex hang modes or failure modes, with the diagnostic that identifies them.
- Real transcripts where `/clodex` caught something a single-model review missed — or
  where it rubber-stamped something it shouldn't have. The skill is encoded from a small
  number of runs and says so; more evidence is the most useful contribution.
- **New cases for the `/rls-audit` tenant matrix.** A leak found by hand that the ten
  standing cases missed is the single most valuable contribution to that skill — a case
  costs one line and then runs forever. `tenant-matrix.md` lists the candidates already
  suspected but not yet written up.

## License

MIT — see [LICENSE](LICENSE).
