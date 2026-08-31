---
name: node-review
description: Review the JS/Node package & build layer in ANY repo — lockfile integrity (drift that installs but fails `npm ci`), package-manager hygiene (single lockfile + packageManager pin), dependency-bump safety (minimal-compatible, clean regen), and test-runner↔runtime compatibility (e.g. vitest 4 vs Node 18). Reports ✅/⚠️/❌; optional --gold emits verified findings as portable training pairs. Use after a dependency bump, before merging a lockfile change, or to audit a repo's supply-chain hygiene.
---

# /node-review — JS/Node package & build-layer review

Four checks against the package/build layer (`package.json`, lockfiles, CI node matrix, a dependency
bump). **No ecosystem dependencies.** This fires on the failure modes that *pass `npm install` but
fail `npm ci`*, ship green because a gate was missing, or break a runtime the bump silently dropped.
For TS *code* → `/ts-review`. For SQL/RLS → `/sql-review`. For privacy/secrets exposure → `/privacy-review`.


**Not a general npm linter.** This fires on the failure modes that *pass install but fail
`npm ci`*, ship green because a gate was missing, or break a runtime the bump silently
dropped. For TS/JS *code* security → `/ts-review`. For SQL and schema → `/sql-review`.

## Usage

```
/node-review <repo-dir>     # full package-layer scan
/node-review                # review the dependency change in this session / `git diff`
/node-review --gold         # append verified findings to .audit/gold-pairs.jsonl (see README)
```

## Severity

| Icon | Level | Meaning |
|------|-------|---------|
| ❌ | error | Will break CI / is broken now. Fix before merge. |
| ⚠️ | warning | Latent drift or hygiene gap. |
| ℹ️ | info | Hardening / convention. |
| ⚪ | skipped | Not applicable. |

---

## Check 1 — lockfile_integrity (fires when a lockfile changed)
- ❌ Lockfile committed after `npm update` / a partial install. **Verify: `npm ci --ignore-scripts`** — if `EUSAGE … not in sync` or `Missing X from lock file`, it's broken. Fix: `rm -rf node_modules package-lock.json && npm install`, then re-verify with `npm ci`.
- ❌ Lockfile change "validated" only by `npm install` or a passing test run — neither proves `npm ci` reproducibility.
- ⚠️ Optional/platform/wasm deps present (`@emnapi/*`, prebuilt binaries) and the lockfile was hand-edited or partial-updated.
- (Same idea for `pnpm-lock.yaml` → `pnpm install --frozen-lockfile`; `yarn.lock` → `yarn install --immutable`.)

## Check 2 — package_manager_hygiene
- ❌ **Two lockfiles committed** (`package-lock.json` + `pnpm-lock.yaml`/`yarn.lock`). Pick canonical from CI + scripts + `workspaces` (not mtime); `git rm` the stray.
- ⚠️ No `packageManager` field → nothing stops a stray install from re-creating a second lockfile. Fix: `npm pkg set "packageManager=<npm|pnpm>@<version>"`.
- ⚠️ A bump done with a different PM than the tracked lockfile (e.g. `pnpm update` in an npm repo) → tracked lockfile silently left stale.

## Check 3 — runtime_compat (fires on a major bump of a test runner / build tool, or an `engines`/CI-matrix change)
- ❌ Major test-tool bump (vitest/jest/etc.) **without checking the CI Node matrix** — a dropped Node version turns the bump red. (e.g. vitest 4.x needs Node ≥20.12 for `node:util` `styleText`; an `18.x` matrix entry fails.)
- ⚠️ Bumped to *latest major* to fix a CVE when a *minimal* patched version exists in a still-compatible line (staying on the older major keeps the supported Node range).
- Fix: pick minimal-compatible (e.g. `vitest@^3.2.6`), **or** drop the unsupported matrix entry **in the same PR** — never leave the contradiction.

## Check 4 — dependency_bump
- ⚠️ Sibling sub-packages (`@vitest/*`, `@babel/*`, etc.) not aligned to the same version as the parent (some require exact-match siblings) → resolution silently pins the old one.
- ⚠️ Caret already covers the patched version but the lockfile still pins the vulnerable one — `npm update <pkg>` (then re-verify Check 1).
- ❌ A dependency flagged by `npm audit` (high/critical) with **no fix available** still in the runtime path (e.g. used to parse user uploads) — flag the exposure and recommend a swap or sandbox, not a silent ignore.
- ℹ️ A CVE-patch PR that drags unrelated transitive churn (sign it wasn't minimal-compatible).

---

## Suggested run

```bash
cat package.json | grep -A40 '"dependencies"'
ls package-lock.json pnpm-lock.yaml yarn.lock 2>/dev/null      # Check 2: one, not many
npm audit --omit=dev                                            # Check 4: highs/criticals + no-fix
npm ci --ignore-scripts >/dev/null 2>&1 && echo "ci OK" || echo "❌ ci out of sync"   # Check 1
grep -rE "node-version|engines" .github package.json 2>/dev/null   # Check 3: matrix vs bumps
```

## Output

Per check: `✅/⚠️/❌ <check> — <finding> (file)`. Then a short must-fix list. A clean check is `✅` —
don't manufacture findings.

## Verify before you assert (and before emitting a gold pair)

Don't claim a lockfile is broken without **running** `npm ci` (or the PM equivalent); don't claim a
runtime break without checking the actual matrix vs the tool's required Node. Confirm against the
real files/commands. If a suspected issue is fine, record it (with `--gold`, as a `false_alarm_of`
negative example).

## `--gold` emission

With `--gold`, after verification append one JSON line per verified finding (and per verified
clear/dismissed claim) to `.audit/gold-pairs.jsonl`, per the gold-pair schema in the repo README. Redact any secret
values (e.g. tokens in a `.npmrc` snippet) in `snippet`. Stamp `repo`/`commit` from git (best-effort).
