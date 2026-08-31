# Treebird Public Skills

Public [Claude Code](https://claude.com/claude-code) skills from the Treebird flock.

Three families: skills that drive *another* coding CLI from inside Claude Code, skills
that audit a repo or review it with deliberately cold eyes, and one that grills you
before you write a hook.

### Cross-model — drive another coding CLI

| Plugin | Command | What it does |
|---|---|---|
| `skill-codex` | `/codex` | Delegate a task to the Codex CLI — model selection, sandbox modes, contract-driven dispatch, and the hang modes that bite in headless runs |
| `skill-clodex` | `/clodex` | Two-model adversarial review under a negotiated contract — each verifies the other's claims, then the fixes split by disjoint file set and land in one build |
| `skill-opencode` | `/opencode` | Dispatch builds and refactors to the opencode CLI across its cloud models, then review what comes back instead of accepting it |

### Audit — review any repo, no ecosystem dependencies

These four detect the DB, ORM, and language from the repo itself. They assume no shared knowledge
tree, no training pipeline, and no particular provider. Run together they're a full sweep of a
TS/Node app: **privacy-review** (what leaks) → **ts-review** (trust-boundary code) →
**sql-review** (DB/RLS in the migrations) → **rls-audit** (DB/RLS as actually deployed) →
**node-review** (supply chain).

`sql-review` reads what the migrations *say*; `rls-audit` proves what the running project
*does*. Neither substitutes for the other.

| Plugin | Command | What it does |
|---|---|---|
| `skill-rls-audit` | `/rls-audit` | Audit a deployed Supabase project for multi-tenant RLS leaks — and *demonstrate* isolation with a two-tenant black-box matrix rather than inspecting it |
| `skill-privacy-review` | `/privacy-review` | A fearless privacy inventory of an app holding sensitive user data — hunt the gap between the privacy you promise and what the code enforces |
| `skill-sql-review` | `/sql-review` | SQL migrations & queries: migration safety, RLS correctness, function security, live privilege drift, concurrency races, blanket grants |
| `skill-ts-review` | `/ts-review` | Security-scoped TS/JS: input validation, subprocess spawning, env leakage, path traversal, file permissions, error discipline, CSP transport coverage |
| `skill-node-review` | `/node-review` | Package & build layer: lockfile integrity (`npm ci` drift), package-manager hygiene, dependency-bump safety, runtime compatibility |

All four report `✅ / ⚠️ / ❌` per check and accept `--gold` — see [Gold pairs](#gold-pairs-gold).

### Cold-eyes review

| Plugin | Command | What it does |
|---|---|---|
| `skill-ux-foolproof-review` | `/ux-foolproof-review` | Run your own onboarding flow as a complete stranger — cold machine, zero credentials, zero tribal knowledge — and log every friction point as a product bug |

### Authoring discipline

| Plugin | Command | What it does |
|---|---|---|
| `skill-hooks-create` | `/hooks-create` | Grill yourself through the portability, blocking, and silent-failure patterns *before* writing an AI-CLI or git hook |

## Requirements

- Claude Code.
- Each cross-model skill shells out to its own CLI — install only the ones you'll use:

  | Plugin | Needs | Verify |
  |---|---|---|
  | `skill-codex`, `skill-clodex` | `codex` CLI + a working login (ChatGPT sign-in, device auth, or API key) | `codex --version` |
  | `skill-opencode` | `opencode` CLI + credentials for at least one provider | `opencode --version` |

  Resolve any errors from those before installing — the skills assume the CLI runs.
- **For `/rls-audit` only:** read access to the Supabase project you're auditing — the
  Supabase CLI linked to it (`supabase link`), the Supabase MCP server, or the SQL editor.
  `psql`, `curl`, and `jq` for the two-tenant matrix.
- The audit, review, and authoring skills (`privacy-review`, `sql-review`, `ts-review`,
  `node-review`, `ux-foolproof-review`, `hooks-create`) need nothing beyond Claude Code and the
  repo you're pointing them at. `/sql-review --live` additionally wants `psql` and a reachable
  `$DATABASE_URL`; without them it reports those checks as skipped rather than failing.

## Installation

### Option 1 — Plugin marketplace (recommended)

```
/plugin marketplace add treebird7/treebird-oss-skills
/plugin install skill-codex@treebird-oss-skills
/plugin install skill-clodex@treebird-oss-skills
/plugin install skill-opencode@treebird-oss-skills
/plugin install skill-ux-foolproof-review@treebird-oss-skills
/plugin install skill-rls-audit@treebird-oss-skills
/plugin install skill-privacy-review@treebird-oss-skills
/plugin install skill-sql-review@treebird-oss-skills
/plugin install skill-ts-review@treebird-oss-skills
/plugin install skill-node-review@treebird-oss-skills
/plugin install skill-hooks-create@treebird-oss-skills
```

Install only what you want — the plugins are independent, with one exception:
`/clodex` calls into `/codex` for pre-flight login, error handling, and the Codex CLI
flag/model mechanics, so `skill-clodex` declares `skill-codex` as a plugin dependency.
Installing `skill-clodex` pulls `skill-codex` in with it.

### Option 2 — Standalone skills

```bash
git clone --depth 1 https://github.com/treebird7/treebird-oss-skills.git /tmp/treebird-oss-skills && \
mkdir -p ~/.claude/skills && \
for s in codex clodex opencode privacy-review rls-audit sql-review ts-review node-review \
         ux-foolproof-review hooks-create; do \
  cp -r /tmp/treebird-oss-skills/plugins/skill-$s/skills/$s ~/.claude/skills/$s; \
done && \
rm -rf /tmp/treebird-oss-skills
```

Drop any skill you don't want from that `for` list. `/rls-audit` ships a companion file
(`tenant-matrix.md`) alongside its `SKILL.md` — the loop copies whole directories, which is
what you want.

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

## `/opencode` — dispatch to opencode

> Have opencode scaffold the CLI skeleton, then check its work.

Two modes: scaffold a project from nothing, or hand it a scoped change in an existing
repo. Either way the skill's position is that **opencode's output is a draft, not a
result** — it dispatches, resumes the session when the work spans turns, and then reads
the diff critically rather than reporting success because the command exited 0.

Its model table carries the same dated-floor caveat as `/codex` — see
[Models and freshness](#models-and-freshness).

## `/ux-foolproof-review` — be the stranger

> Foolproof our install flow before we publish it.

Most onboarding review imagines a new user. This one *is* one: cold environment, no repo
checkout, no env vars, no secrets manager, no memory of the design discussions. Every
place the reviewer has to guess, re-read, or reach for knowledge they were never given is
filed as a product bug — not a docs nit.

The honest part: if you can't actually fake cold — an already-logged-in CLI, a cached
credential, a machine-wide config leaks in — the skill makes you **name what leaked**
rather than quietly benefit from it.

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
- **Write policies** — the *effective* write check is `coalesce(with_check, qual)`, since
  PostgreSQL reuses `USING` when `WITH CHECK` is omitted on `UPDATE`/`ALL`. The leak is a
  `USING` expression that reads fine but checks weakly: `user_id = auth.uid()` reused as a
  write check still lets you rewrite `tenant_id`, because you stay the owner. Read-only test
  suites never catch any of it.
- **Permissive-OR** — a narrow policy does not constrain a broad one beside it, so a
  `DROP POLICY IF EXISTS` with a stale name is a silent no-op that leaves the leak live.
- **The surfaces nobody audits** — `SECURITY DEFINER` RPCs (anon-callable by default),
  column-level grant breadth, public storage buckets and unpinned object paths, the
  `supabase_realtime` publication, and service-role keys in client bundles.
- **Existence oracles** — `count=exact` returning a positive number with an empty body, and
  unique-constraint violations whose error text confirms another tenant's row exists.

- **BOLA / IDOR** — the black-box matrix substitutes A's row IDs, RPC parameters, embeds,
  storage paths, foreign keys, slugs, and GraphQL IDs as B, across every supported operation
  plus bulk and nested variants. It also separates tenant isolation from same-tenant
  ownership and role checks.

And the part that separates it from every checklist: **Check 7, the two-tenant matrix.**
Everything else *inspects*; this *demonstrates*. Two real tenants, fifteen cases, anon key only
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
- **Check 8 exists because audits decay.** It proposes a CI gate — six allowlist-diffed catalog rules that
  must each return zero rows — so the table added next month can't quietly reintroduce
  what you just fixed.

## `/sql-review` — migrations, RLS, and the grants nobody migrated

> Review this migration before we deploy it.

Seven checks over a `.sql` migration, a Prisma/Drizzle schema, or a raw query. The two that earn
their keep:

- **Function security triaged by *identity source*, not by grant.** The question isn't "does this have a REVOKE" — it's whose identity the function acts on. A `SECURITY DEFINER` function that derives identity from `auth.uid()` is safe even when PUBLIC-callable; one that takes a `user_id` **parameter** and trusts it is a cross-user read/write with RLS bypassed by design. Grading those the same way buries the real hole under hygiene noise, so the skill refuses to.
- **Privilege drift (`--live`).** Drift isn't only columns. A "hardening" migration whose `REVOKE` never ran, or whose `DROP POLICY IF EXISTS` names a policy that doesn't exist live, reads as done and leaves the hole open. The skill queries actual grants and live policy names before believing the migration.

It also catches the `CREATE OR REPLACE` silent revert — redefining a function from an *older* base
than the deployed one drops any guard added in between, with no merge, no conflict, and a green
test suite.

## `/ts-review` — only where TS crosses a trust boundary

> Review this runner module.

Seven checks, and a scope heuristic that skips pure logic: it fires only on code touching
`child_process`, `fs` with a user-derived path, `process.env`, or a user string headed for a path,
subprocess, shell, regex, or query. `tsc` and ESLint already own style — this owns the exploit paths.

Check 7 is the one people don't expect: a CSP granting `https://<host>` for a host also used as a
**websocket** backend, with no `wss://` twin. Browsers don't scheme-match the two, so the socket is
blocked, the client library retries quietly forever, and nothing throws. It presents as "realtime is
just flaky" until someone reads a `securitypolicyviolation` event.

## `/node-review` — the package layer

> Check this dependency bump before I merge it.

Four checks on the failure modes that pass `npm install` and fail `npm ci`: lockfile drift committed
after a partial update, two lockfiles in one repo with no `packageManager` pin to stop a third, a
major test-runner bump that silently drops a Node version still in the CI matrix, and a CVE patch
taken at latest-major when a minimal-compatible patch exists in the supported line.

The skill refuses to claim a lockfile is broken without running `npm ci` — a review that guesses at
reproducibility is worth nothing.

## `/privacy-review` — the gap between promise and enforcement

> Run a privacy review before this goes public.

For an app that holds people's private writing, notes, journals, health data. The goal
isn't a compliance checklist and it isn't reassurance — it's finding where the code's
actual behavior diverges from the privacy the product promises, while there's still time
to close the gap for free.

## `/hooks-create` — grill before you write

> Add a PreToolUse hook that blocks `npm` commands.

Hooks fail in a specific, nasty way: they run on every action, they run outside your
attention, and when they break they usually break *silently* — the log line never
written, the block that never fired, the path that only resolves on the author's machine.

So this skill refuses to write hook code first. It walks the surface (Claude Code /
Copilot CLI / Codex CLI / opencode / git), the event, the single job the hook does, the
failure mode, and the portability traps — then writes. Every skipped question is a
bug-shaped opening.

## Gold pairs (`--gold`)

The four audit skills accept `--gold`. When set, each **verified** finding — and each verified
`clear` — is appended as one JSON line to `.audit/gold-pairs.jsonl` in the reviewed repo. They're
review-decision training examples: portable, schema-documented, and yours to keep.

```jsonc
{
  "schema": "audit-kit/gold-pair@1",
  "skill": "sql-review",             // sql-review | privacy-review | ts-review | node-review
  "check": "rls_correctness",        // the check id that produced it
  "severity": "error",               // error | warning | info | critical | high | medium | clear
  "repo": "owner/name",              // best-effort from `git remote`; "" if none
  "commit": "81b2c7b",               // short HEAD at review time
  "path": "prisma/schema.prisma:42", // file:line (or file: if line unknown)
  "lang": "sql",                     // detected language/ORM tag
  "snippet": "CREATE TABLE ...",     // the reviewed code, secrets redacted
  "finding": "table created without ENABLE ROW LEVEL SECURITY",
  "fix": "ALTER TABLE x ENABLE ROW LEVEL SECURITY; + per-op policies",
  "verified": true,                  // only verified findings are emitted
  "false_alarm_of": null             // if set, names a dismissed claim — a negative example
}
```

**Why `verified` matters:** a gold pair is only worth keeping if the finding was confirmed against
the actual source rather than pattern-matched in the abstract. Every audit skill runs a verify step
and emits a pair only on a confirmed verdict — including confirmed *non*-findings
(`severity: "clear"`, or `false_alarm_of` set), which are the most valuable negative examples.

The file carries no secrets — only the snippet, the verdict, and the fix — and the emitting skill
must redact secret values from `snippet` (keys, tokens, connection strings → `‹redacted›`). That
makes it safe to hand back after reviewing someone else's repo.

## Models and freshness

`/codex`, `/clodex`, and `/opencode` carry a dated model table rather than relying on memory, because model
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
- **New cases for the `/rls-audit` tenant matrix.** A leak found by hand that the fifteen
  standing cases missed is the single most valuable contribution to that skill — a case
  costs one line and then runs forever. `tenant-matrix.md` lists the candidates already
  suspected but not yet written up.
- Failure modes the audit skills missed on a real codebase, and the check that would
  have caught them.
- New checks for `/sql-review` on dialects beyond Postgres — MySQL and SQLite currently
  fall through as `⚪ skipped` on the RLS and function-security checks.

## License

MIT — see [LICENSE](LICENSE).
