---
name: ts-review
description: Security-scoped TypeScript/JavaScript review for ANY repo — input validation, subprocess spawning, env/secret leakage, path traversal, file permissions, and error discipline. NOT a style linter (tsc + ESLint cover that); fires only on code crossing a trust boundary. Reports ✅/⚠️/❌ per check; optional --gold emits verified findings as portable training pairs. Use after writing or when handed TS/JS that accepts user input, spawns processes, writes files, or touches process.env.
---

# /ts-review — security-scoped TypeScript review

Seven checks against TS/JS that crosses a trust boundary. **No ecosystem dependencies.** Not a general
linter — `tsc` + ESLint cover style and types. This fires only on code that accepts user input,
spawns subprocesses, writes files, or reads `process.env`.

## Usage

```
/ts-review <path>      # a file, or a dir (recurse *.ts/*.tsx/*.js/*.mjs)
/ts-review             # review TS/JS changed this session or in `git diff`
/ts-review <path> --gold   # append verified findings to .audit/gold-pairs.jsonl (see README)
```

## Scope heuristic (skip pure logic)

Run the checks only when the file touches at least one of:
- `child_process` (`spawn`, `exec`, `execFile`, `fork`)
- `fs` / `fs/promises` with a user-derived path
- `process.env` (reading or passing through)
- a function taking a user-typed string into a path, subprocess arg, shell, regex, or DB query

No security-relevant surface → `⚪ not applicable`.

## Severity

| Icon | Level | Meaning |
|------|-------|---------|
| ❌ | error | Direct exploit path. Fix before shipping. |
| ⚠️ | warning | Likely issue / hardening gap. |
| ℹ️ | info | Hygiene, naming, missing assertion. |
| ⚪ | skipped | Not applicable to this file. |

---

## Check 1 — input_validation
Fires when a caller string reaches a filesystem, subprocess, DB, or regex.
- ❌ Allowlist regex without `^...$` anchors (substring match → attacker appends arbitrary suffix).
- ❌ Denylist where an allowlist fits (`if (!input.includes('..'))` misses `%2e%2e`, null bytes, unicode).
- ❌ User string into `new RegExp(userInput)` without escaping (ReDoS / injection).
- ❌ Validator returns the raw string instead of a parsed/typed structure when it's then passed to `spawn`/a query.
- ⚠️ Allowlist regex with no length cap (`{1,N}`); no null-byte rejection before an fs call; `\S+`/`.+` inside a char class (too permissive); missing `typeof input !== 'string'` guard.
- ℹ️ Validator returns `void` (caller gets no typed result); error message doesn't say what was expected.

## Check 2 — spawn_hygiene
Fires on any `child_process` use.
- ❌ `spawn`/`exec` with `shell: true` **and** any user-derived content in command or args.
- ❌ `exec(template)` with user input (exec defaults to a shell); string-form `spawn('cmd a b')` with user input.
- ❌ No timeout on a subprocess consuming user-controlled data or network.
- ⚠️ `stdio: 'inherit'` on a subprocess handling untrusted input; `spawnSync` with no error check; missing `child.on('error')` (unhandled EACCES/ENOENT crash); `cwd` from user input without validation.
- ℹ️ Timeout is a magic number — name it; no hard-coded `PATH` in a Node env that spawns external binaries (dev-mode brittleness, not security).

## Check 3 — env_isolation
Fires on `spawn`/`exec`/`fork`, or any function building an env dict for a child, or env reaching a sink.
- ❌ `spawn(..., { env: process.env })` (or `{ ...process.env, X }`) on a child handling untrusted input.
- ❌ `process.env` passed to a logger, telemetry sink, or error reporter.
- ❌ API keys/secrets read from `process.env` and included in a child's env without filtering.
- ⚠️ Env builder that doesn't explicitly enumerate keys (`{ ...defaults, ...extras }`); no assertion between env build and spawn; denylist missing a known-sensitive name. **Generic sensitive set to check** (extend per project): `*_SECRET`, `*_TOKEN`, `*_KEY`, `*_PASSWORD`, `DATABASE_URL`, `AWS_`, `OPENAI_`, `ANTHROPIC_`, `GOOGLE_`, `GITHUB_`, `STRIPE_`, `RESEND_`.
- ℹ️ Comment-only "don't add secrets here" with no code guard; each consumer redefines its own denylist instead of a shared one.

## Check 4 — path_safety
Fires on any `fs` call with a user-derived path segment.
- ❌ `path.join(root, userPath)` without an `path.isAbsolute` rejection (absolute `userPath` escapes `root`).
- ❌ `path.resolve` result used without a `startsWith(path.resolve(root) + path.sep)` containment check.
- ❌ Writes to sensitive paths (`.git/hooks/`, `.env`, `id_rsa`, `authorized_keys`) not explicitly blocked.
- ❌ User-controlled symlink target followed without `fs.realpath` + containment re-check.
- ⚠️ `path.normalize` as the only defence (strips `..` but doesn't guarantee containment); `startsWith('..')`-only check (misses `a/../../b`); `fs.access`-then-`readFile` TOCTOU.
- ℹ️ Containment check missing `path.sep` (POSIX-only); `fs.mkdir` without `recursive: true` where nesting is expected.

## Check 5 — permissions_hygiene
Fires on `fs.writeFile`/`mkdir`/`chmod`.
- ❌ A secret/token/decrypted value written with default mode (0o644 → world-readable); private key / export written without explicit `mode: 0o600`; sandbox dir created without explicit mode (relies on umask).
- ⚠️ Marker/sentinel file not `mode: 0o444`; no test asserting `stat.mode & 0o077 === 0` on sensitive writes; `chmod` after `writeFile` (race window) instead of `mode:` in options.
- ℹ️ Magic octal — name it (`const SECRET_MODE = 0o600`).

## Check 6 — error_discipline
Fires on `try/catch`, async functions, validation gates.
- ❌ Validation outside the try/catch when the contract is "always persist a result" (throw → no result, monitoring misses it); empty `catch {}` on a trust-boundary op; floating promise (returned promise neither awaited nor chained); `throw <string>` / `throw null`.
- ⚠️ `catch (err)` that swallows without log/rethrow; `catch { throw new Error('failed') }` dropping `cause`/stack; error message with no layer/why context; `.then(x, y)` instead of `.catch(y)`.
- ℹ️ Thrown `Error` could carry `cause: innerErr`; no error-class hierarchy.


## Check 7 — csp_transport_coverage
Fires when the diff touches a Content-Security-Policy (`next.config.*`, `vercel.json`, middleware,
header config) **or** introduces a websocket consumer (`new WebSocket(`, a realtime channel
subscription, socket.io, any `wss://` URL).
- ❌ `connect-src` (or the governing `default-src`) grants `https://<host>` for a host the app also uses as a **websocket** backend, with no `wss://<host>` twin. Browsers do not scheme-match the two: the socket is blocked, and client libraries retry quietly forever. Nothing throws — only a `securitypolicyviolation` event fires — so this reads as "realtime is just flaky" for as long as nobody checks.
- ❌ A CSP edit removes an existing `wss:`/`ws:` source while websocket consumers for that host remain in the code.
- ⚠️ A websocket subscription with no surfaced status callback (a bare `.subscribe()`) **and** a fallback path that masks transport death — together they make the failure above undetectable in production.
- ⚠️ A new realtime host added in code with no CSP change in the diff. It may be covered by `'self'` — verify, don't assume.
- ℹ️ Same-origin websockets are covered by `'self'` and need no `wss:` twin — note it rather than flag it.

---

## Output

```
TS REVIEW — <file or "inline">
Check 1 input_validation … ✅|⚠️ N|❌ N      Check 4 path_safety ………… …
Check 2 spawn_hygiene …… …                  Check 5 permissions_hygiene …
Check 3 env_isolation …… …                  Check 6 error_discipline …… …
Check 7 csp_transport_coverage … …

FINDINGS
❌ [spawn_hygiene] shell:true with user input — src/runner.ts:42 → validate against allowlist, spawn('node',[scriptFor[task]])
⚠️ [path_safety] no containment after path.resolve — src/runner.ts:68 → if(!full.startsWith(resolve(root)+sep)) throw
```
A clean applicable check is `✅`; no surface is `⚪`. Don't manufacture findings to fill the report.

## Verify before you assert (and before emitting a gold pair)

Confirm every ❌/⚠️ against the **actual line** and its surroundings — check whether a guard exists
upstream (a validator, a wrapper, a framework default) before calling it a finding. If a suspected
issue is actually safe, record it (and with `--gold`, as a `false_alarm_of` negative example). State
confidence; separate confirmed from suspected. A wrong "RCE/critical" handed to a repo owner burns trust.

## `--gold` emission

With `--gold`, after verification append one JSON line per verified finding (and per verified
clear/dismissed claim) to `.audit/gold-pairs.jsonl`, per the gold-pair schema in the repo README. **Redact secret values**
in `snippet`. Stamp `repo`/`commit` from git (best-effort). One (anti-pattern → finding → fix) = one pair.
