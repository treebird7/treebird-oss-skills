---
name: sql-review
description: Review SQL migrations and queries in ANY repo for safety, RLS correctness, function security, privilege drift, concurrency races, blanket grants, and query quality. DB/ORM-agnostic. Reports ✅/⚠️/❌ per check; optional --gold emits verified findings as portable training pairs. Use when reviewing a .sql migration, an ORM schema (Prisma/Drizzle/Kysely), or a raw query — especially before deploying schema changes to a multi-tenant or user-data app.
---

# /sql-review — portable SQL & schema review

Runs static checks against SQL (or ORM schema/query code) and reports `✅ / ⚠️ / ❌` per check.
**No ecosystem dependencies** — detects DB and ORM from the repo, never assumes a knowledge tree,
training pipeline, or a specific provider.

## Usage

```
/sql-review <path>            # review a file (.sql migration, schema.prisma, *.ts query module)
/sql-review                   # review SQL/schema changed in the current session or `git diff`
/sql-review <path> --live     # also diff against a reachable DB (psql/$DATABASE_URL) if available
/sql-review <path> --gold     # append verified findings to .audit/gold-pairs.jsonl (see README)
```

## Step 0 — detect the stack (do this first, cheaply)

```bash
ls prisma/schema.prisma drizzle.config.* 2>/dev/null      # ORM?
grep -rilE "createPolicy|enable row level|USING \(|\$queryRaw|\$executeRaw" . | grep -v node_modules | head
```
- Pure `.sql` with DDL (`CREATE/ALTER/DROP/CREATE POLICY/CREATE FUNCTION`) → **migration mode**.
- Prisma/Drizzle schema → translate models to the equivalent table checks; RLS lives in raw SQL
  migrations, so flag tenant tables that have **no** corresponding `ENABLE ROW LEVEL SECURITY`.
- Inline query / snippet → **query mode** (Checks 6 + applicable RLS/function checks).
- Note the dialect (Postgres/MySQL/SQLite). Some checks are Postgres-specific — mark `⚪ skipped` elsewhere.

**On role names.** Checks below name `anon` / `authenticated` / `service_role` — the conventional
Postgres roles in a PostgREST-style stack (Supabase and similar). Substitute your own
anonymous / logged-in / privileged role names; the reasoning is unchanged.

## Severity

| Icon | Level | Meaning |
|------|-------|---------|
| ❌ | error | Correctness, security, or data-loss risk. Fix before deploy. |
| ⚠️ | warning | Likely issue or heuristic concern. Review. |
| ℹ️ | info | Hygiene / convention. |
| ⚪ | skipped | Not applicable (wrong dialect, no live conn, etc.). |

---

## Check 1 — migration_safety
- ❌ `ALTER TABLE ... DROP COLUMN` / `ALTER COLUMN TYPE` with no `-- ROLLBACK:` note (irreversible + table rewrite).
- ❌ Non-IMMUTABLE function (`now()`, `CURRENT_DATE`, `current_setting()`) in an index expression (apply fails) or `CHECK` constraint (write-time-only trap → move to a `BEFORE` trigger).
- ❌ Adding a `UNIQUE` constraint when existing `INSERT` paths elsewhere lack `ON CONFLICT` / unique-violation handling → those paths start throwing raw constraint errors. Grep other `INSERT INTO <table>` sites.
- ❌ **`CREATE OR REPLACE FUNCTION` that drops a guard added by a *later* migration than the one being mirrored (silent revert).** When a migration redefines a function/trigger/RPC, the new body must be a **superset of the object's latest live definition** — not of the original. Grep every prior migration touching `function public.<name>` (or read `\sf public.<name>` with `--live`), diff against the newest, and flag any guard, branch, or `RAISE` present there and absent here. `CREATE OR REPLACE` overwrites with no merge and no conflict, so a reverted guard is invisible and the suite stays green unless a regression test replays that guard's scenario. Require (a) the new body preserves every prior guard, and (b) each guard has a pinning test that fails if it's dropped. Prefer an *additive* migration (new trigger / CHECK) over a full-body rewrite when the change is orthogonal.
- ⚠️ Missing idempotency guards: `CREATE TABLE`/`INDEX` without `IF NOT EXISTS`; `DROP ...` without `IF EXISTS`; `CREATE FUNCTION` that could be `CREATE OR REPLACE`.
- ⚠️ `NOT NULL` added to an existing column with no `DEFAULT` (lock + fails on existing rows); new constraint without `NOT VALID` then `VALIDATE` (validation scan locks).
- ⚠️ `CREATE INDEX` without `CONCURRENTLY` on a large/hot table (write lock).
- ℹ️ File not named `<timestamp>_<description>`; missing `-- ROLLBACK:` / `-- Description:` header.

## Check 2 — rls_correctness (Postgres; the highest-value check for multi-tenant apps)
- ❌ `CREATE TABLE` on a client-reachable/tenant table with no `ENABLE ROW LEVEL SECURITY` in the same migration.
- ❌ Single `FOR ALL` policy — split into per-operation (SELECT/INSERT/UPDATE/DELETE); they have different predicates.
- ❌ Policy derives identity from a **request body / function parameter / variable** instead of the session identity (`auth.uid()`, `current_setting('app.user_id')`, JWT claim).
- ⚠️ `INSERT`/`UPDATE` policy missing `WITH CHECK` (the row can be written into a state the policy would forbid reading).
- ⚠️ `USING (true)` / `USING (1=1)` on a tenant/sensitive table → any caller reads every row.
- ⚠️ Authenticated-but-not-owner-scoped: `USING (auth.role() = 'authenticated')` / `USING (auth.uid() IS NOT NULL)` on a per-user table → any logged-in user reads all rows. Needs an owner predicate.
- ⚠️ Permissive UPDATE policy on a **shared/multi-party row** coexisting with a table-wide `UPDATE` grant: RLS scopes rows, not columns → a non-owner who satisfies the row predicate can rewrite *any* column. Fix with column-level `GRANT UPDATE(col,…)`.
- ⚠️ Narrowing a content table while leaving its **association/identity** tables (likes, follows, handles) at `USING (true)` → the relationship is reconstructable by JOIN. Treat them as one anonymity surface.
- ⚠️ `auth.uid()` called inline instead of cached `(SELECT auth.uid())` (perf at scale); missing index on RLS-predicate columns.
- ℹ️ Policy name off-convention (`<table>_<op>_<scope>`).

## Check 3 — schema_drift (`--live` only)
If a DB is reachable (`$DATABASE_URL` + `psql`, or the ORM's introspect): compare declared columns/
types/constraints/**policies + grants** against deployed. Flag:
- ❌ A tightening migration whose `DROP POLICY IF EXISTS "<name>"` doesn't match the **live** policy name → silent no-op, the broad policy survives (permissive-OR keeps the table open). Verify live names before apply.
- ⚠️ Declared-vs-deployed column/type/nullable drift; grants broader live than the migration implies.

### Sub-check 3a — privilege_drift (the grant the migrations don't show)

Drift is not just columns — **EXECUTE and table grants drift too**, and that's where security
regressions hide. A function can be hardened on prod by a manual console fix no migration records;
worse, a migration's `REVOKE` may never actually have run. Compare *declared* grants to *live* grants:

```bash
# Anonymous / PUBLIC EXECUTE on functions (should be empty for locked-down fns)
psql "$DATABASE_URL" -c "
  SELECT routine_name, grantee FROM information_schema.routine_privileges
  WHERE routine_schema='public' AND grantee IN ('anon','PUBLIC');"

# Table grants to the anonymous / logged-in roles
psql "$DATABASE_URL" -c "
  SELECT table_name, grantee, privilege_type FROM information_schema.role_table_grants
  WHERE table_schema='public' AND grantee IN ('anon','PUBLIC');"

# Column-level UPDATE breadth — does the role hold UPDATE on more columns than the app writes?
psql "$DATABASE_URL" -c "
  SELECT column_name FROM information_schema.column_privileges
  WHERE table_schema='public' AND table_name='<table>'
    AND grantee='authenticated' AND privilege_type='UPDATE' ORDER BY 1;"

# Live policy names — run BEFORE approving a DROP POLICY IF EXISTS tightening migration
psql "$DATABASE_URL" -c "
  SELECT policyname, cmd, qual FROM pg_policies
  WHERE schemaname='public' AND tablename='<table>';"
```

- ❌ A function the migration intends to lock down **still has anonymous/`PUBLIC` EXECUTE live** — the REVOKE never ran, or was never written. This is the class where a "hardening" migration reads as done and the hole is open.
- ❌ A live grant no migration in the tree accounts for (untracked manual change — reconcile it into a migration).
- ❌ **Column-UPDATE breadth exceeds the app's write surface**: the logged-in role holds `UPDATE` on columns the app never writes, on a table whose UPDATE policy a non-owner can satisfy → content-tamper vector (see Check 2). Fix: `REVOKE UPDATE ON <table> FROM authenticated; GRANT UPDATE(<only the columns the app writes>) ON <table> TO authenticated;`
- ❌ **A `DROP POLICY IF EXISTS` target that matches no live `policyname`** → the DROP silently no-ops and the broad policy survives. The tightening migration is a false fix.
- ⚠️ A declared REVOKE/GRANT already satisfied live is a no-op (informational — confirms convergence); one *not* yet applied is "lockdown not deployed".
- ⚪ `schema_drift: not run (no --live flag or no reachable DB)`.

### Sub-check 3b — migration_applied_correctly

A migration file can be correct, committed, reviewed — and never actually applied, or applied to the
**wrong database**. Both fail silently: the code calling it ships, the file exists, nothing errors
until a user hits the missing object. Run whenever a timestamped migration filename is available.

If your migration tool tracks applied state (`supabase migration list`, `prisma migrate status`,
`atlas migrate status`, a `schema_migrations` table), read it first — one call answers "is every
local migration applied here":

```bash
psql "$DATABASE_URL" -c \
  "SELECT version FROM schema_migrations ORDER BY version DESC LIMIT 20;"   # adjust to your tool's table
```

- ❌ The migration appears applied on a **different** database than intended (wrong target).
- ❌ The migration appears on **multiple** databases (double-applied — conflicting or duplicated state).
- ⚠️ **Local migration present, remote entry absent** — written but never applied. Especially dangerous when app code already calls the objects it creates; check whether anything in the diff invokes them.
- ⚠️ Timestamp matches but the name differs (a rename after apply — the tool may re-run or skip it).
- ⚪ No reachable DB, or inline SQL with no filename to key on → report skipped and recommend manual verification.

**A status check against one database cannot detect the wrong-target case** — it can only see the
database it's pointed at. If status is clean but behavior says otherwise, check the other environments
by hand before concluding the migration ran.

## Check 4 — function_security (any `CREATE [OR REPLACE] FUNCTION`)

The highest-value check, and the one policy review misses. RLS protects *table access* — it does
nothing about who may *execute a function*, and a `SECURITY DEFINER` function bypasses RLS by design.

Parse each function: name, schema, args (a user-id-shaped param?), `LANGUAGE`,
`SECURITY DEFINER|INVOKER`, `SET search_path`, body. Then triage before grading.

### Triage first, by identity source

The question is **not** "does it have a REVOKE" — it's **whose identity the function acts on, and
whether an arbitrary caller can reach a privileged action.** Identity source sets severity; REVOKE
presence is downstream.

| Bucket | Signature | Verdict |
|---|---|---|
| **1 · param-identity** | Takes a `user_id`/target **param** and uses it in WHERE/UPDATE/INSERT with **no `caller == target` check** | 🔴 **CRITICAL** — cross-user read/write; RLS is bypassed |
| **2 · unguarded privileged write** | DEFINER + INSERT/UPDATE on a sensitive table (credits, balance, subscription, role), no caller authz, EXECUTE reachable by the anonymous or logged-in role | 🔴 **CRITICAL** — forgery / privilege escalation |
| **3 · session-identity-internal** | Identity derived **solely** from `(SELECT auth.uid())` (or equivalent session claim); every read/write self-scoped; anonymous callers get 0 rows | 🟢 **SAFE** even if PUBLIC-callable. Missing REVOKE is **hygiene**, not a breach |
| **4 · trigger / event fn** | `RETURNS TRIGGER`, fired by table events, not reachable as an RPC endpoint | ⚪ **N/A** on this axis. Still needs the `search_path` check |

**Triage rule — do not flood the report.** Missing-REVOKE is an ℹ️ hygiene finding. A ❌ critical is
**Bucket 1 or 2 only**. Escalating every Bucket 3/4 function because a REVOKE is absent buries the
real holes, which is the failure mode this rule exists to prevent.

**Cross-migration nuance:** a later `CREATE OR REPLACE` with the same name **and argument types**
supersedes an earlier vulnerable definition (parameter *names* don't affect identity). Always review
the **latest / deployed** definition, not the first occurrence. With `--live`, confirm which one is
actually deployed.

### ❌ Error-level
- **`SECURITY DEFINER` without `SET search_path`** — search-path hijack → privilege escalation. Needs `SET search_path = pg_catalog, public` (pg_catalog first).
- **`SET search_path` placing `"$user"` or a mutable/untrusted schema ahead of `pg_catalog`** — the same hijack, disguised.
- **Param-identity with no auth guard** (Bucket 1) — a DEFINER fn takes `p_user_id` / `target_user` / similar and uses it in a `WHERE`/`UPDATE`/`SELECT` without comparing it to the session identity. Any caller passes any id. Fix: self-guard (`IF (SELECT auth.uid()) IS DISTINCT FROM p_user_id AND COALESCE((SELECT auth.role()),'') <> 'service_role' THEN RAISE …`) — or, for server-authoritative writes (minting balance, payouts), lock to the privileged role and stop trusting the param.
- **No matching REVOKE — severity by bucket.** Postgres grants `EXECUTE` to `PUBLIC` by default, so a new function with no `REVOKE … FROM PUBLIC` is callable by anyone the API exposes. ❌ only for Bucket 1 or 2; ℹ️ for Bucket 3; N/A for Bucket 4.
- **`REVOKE … FROM anon` without also revoking `PUBLIC`** — a no-op. The anonymous role inherits EXECUTE through `PUBLIC`. The pattern must be `FROM PUBLIC, anon`.
- **Dynamic SQL injection** — `EXECUTE format(...)` interpolating identifiers or values not passed via `USING` (or without `%I`/`%L`), inside a DEFINER fn. Privileged injection surface.
- **Security-config DML inside a function** — `ALTER ROLE`, `GRANT`, `CREATE POLICY`, `ALTER … OWNER` in a body callable by non-superusers is a privilege-escalation gadget.

### ⚠️ Warning-level
- **REVOKE/GRANT signature mismatch** — a `REVOKE`/`GRANT ON FUNCTION public.foo(<sig>)` whose `<sig>` matches no `CREATE FUNCTION` in the file. When a signature changes (a param added or dropped), the old grant lines dangle and the *new overload* ships ungranted-or-unrevoked — a silent way to leave an old vulnerable overload callable.
- **Unnecessary `SECURITY DEFINER`** — the body touches no RLS-protected table and needs no owner privilege. Prefer `SECURITY INVOKER`.
- **Reads a sensitive schema without caller filtering** — selects from `auth.users`, `pg_authid`, `pg_roles`, `information_schema`, or app tables holding payments/secrets, with no session-scoped predicate.
- **`SECURITY DEFINER` on a validation-only trigger function** — no privileged need; if kept for convention, ensure `SET search_path` is present.

### ℹ️ Info-level
- A read-only DEFINER fn returning user data could document its PII surface (which sensitive columns it exposes) for data-classification review.

## Check 5 — grant_hygiene
- ⚠️ `GRANT ... ON ALL TABLES IN SCHEMA public TO anon, authenticated` (future backend-only tables auto-exposed).
- ⚠️ `ALTER DEFAULT PRIVILEGES ... GRANT ... TO anon/public` (every future object becomes reachable).
- ⚠️ `GRANT ... TO PUBLIC` (every role inherits, incl. service/admin).
- ❌ `ALTER TABLE ... DISABLE ROW LEVEL SECURITY` / `NO FORCE` on a client-exposed table.

## Check 6 — query_review (raw queries / ORM `$queryRaw` etc.)
- ❌ String-concatenated/interpolated user input into SQL → injection. Require parameterized (`$1`, prepared, tagged ``sql` ` ``).
- ⚠️ Tenant table queried with no tenant predicate in app code (the "forgot the `where ownerId`" leak). For ORMs, flag `findUnique/findMany` on a tenant model whose `where` lacks the owner key.
- ⚠️ `SELECT *` returning encrypted/sensitive columns into a response unfiltered; N+1 in a loop; missing `LIMIT` on an unbounded user-facing list.

## Check 7 — concurrency_safety

Check-then-write races in functions performing conditional writes. These pass every single-threaded
test and fail only under real concurrency, which is why they ship.

### ⚠️ Warning-level
- **Check-then-INSERT without serialization** — `IF NOT EXISTS (…) THEN INSERT`, or `SELECT` a candidate then `INSERT`, with no advisory lock, no `ON CONFLICT`, and no unique index that fully covers the race. Two concurrent calls both pass the check and both write.
- **A partial-unique index assumed to catch a race it can't** — a `UNIQUE … WHERE` or `LEAST/GREATEST` expression index covering only one race axis (A↔B) while another (A→B1 vs A→B2) slips through. Name the uncovered axis.
- **Count-then-act rate limit** — `SELECT count(*) …; IF count >= N THEN RAISE` then insert, unlocked. Two concurrent calls both read `N-1`.

### ❌ Error-level
- **Advisory locks acquired in non-canonical order** — a fn locking two keys in call order rather than sorted order. Two transactions sharing a participant deadlock (T1 holds A wants B; T2 holds B wants A). Fix: acquire via `LEAST(...)` then `GREATEST(...)`.

---

## Output

For each applicable check, one line: `❌/⚠️/ℹ️/⚪ <check> — <finding> (file:line)`. Then a short
summary: counts by severity + the top 3 must-fix. **Do not** invent findings to fill the report;
`✅ <check> — clear` is a valid and useful result.

## Verify before you assert (and before you emit a gold pair)

Every ❌/⚠️ must be confirmed against the **actual source line**, not pattern-matched in the abstract.
Read the surrounding code; check whether a guard exists elsewhere (a later `OR REPLACE`, a wrapper, an
app-level scope). If a suspected finding turns out safe, record it as `clear` (and, with `--gold`, as
a `false_alarm_of` negative example). Always review the **latest/deployed** definition of a redefined
object, not the first occurrence.

## `--gold` emission

When `--gold` is set, after verification append one JSON line per **verified** finding (and per
verified `clear`/dismissed claim) to `.audit/gold-pairs.jsonl`, matching the gold-pair schema in the repo README.
Redact any secret values in `snippet` (keys, tokens, connection strings → `‹redacted›`). Stamp `repo`
from `git remote get-url origin` (best-effort) and `commit` from `git rev-parse --short HEAD`.
