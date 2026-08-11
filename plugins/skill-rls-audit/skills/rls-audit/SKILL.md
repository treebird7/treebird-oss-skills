---
description: >
  Audit a deployed Supabase / Postgres project for multi-tenant row-level-security leaks —
  policy shape, tenant-claim provenance, the bypass surfaces RLS never covers (views,
  SECURITY DEFINER functions, grants, storage, realtime), existence oracles, and a
  two-tenant black-box proof. Use before onboarding a second tenant, before a security
  review, or after adding any table, view, RPC, bucket, or realtime subscription.
name: rls-audit
---

# /rls-audit — Multi-Tenant RLS Leak Audit

Run 8 checks against a **deployed** Supabase project to prove tenant isolation actually
holds, and report ✅/⚠️/❌ per check.

**This is not a migration reviewer.** It reviews the *running system*, on the assumption
that the migration tree lies — because grants get hand-edited in the SQL editor, tightening
migrations silently no-op, and the table added last month never got a policy.

## When to Use

- **Before opening a multi-tenant app to a second tenant.** One tenant cannot leak to
  itself; every bug here is dormant until customer #2 signs up.
- **Before a security review, pen test, or customer security questionnaire.**
- **After adding** a table, a view, an RPC, a storage bucket, or a realtime subscription —
  the leak is almost never in the table you were thinking about.
- **After any report** of a user seeing data that wasn't theirs.
- **Periodically** on a live project, because deployed state drifts from the migrations.

## Usage

```
/rls-audit                       # full audit against the linked project
/rls-audit <schema>              # scope to one schema (default: every exposed schema)
/rls-audit --static              # catalog + source only; skip the live tenant-B matrix
/rls-audit --matrix              # run ONLY Check 7 (the two-tenant black-box proof)
```

Needs a linked project (`supabase link`) or equivalent read access. Without one, run
`--static` and report Checks 3, 7, 8 as ⚪ skipped — do **not** silently downgrade to a
source-only review and call it an audit.

### Resolve the exposed schemas first

Every query below filters on schema. **Resolve the set once, at the top of the run, and reuse
it** — the audit claims to cover every exposed schema, and a query hardcoded to `public`
silently skips a `SECURITY DEFINER` RPC or an unprotected table living in `api`, `app`, or a
versioned `v1` schema.

```sql
-- PostgREST's own view of what it serves (preferred — it is the actual config)
select current_setting('pgrst.db_schemas', true) as exposed;
```

If that returns NULL (not set at the role level), read `db-schemas` from Dashboard →
Settings → API and pin it explicitly for the rest of the session:

```sql
set rls_audit.exposed_schemas = '{public}';   -- replace with the real list
```

Then every query uses `n.nspname = any (current_setting('rls_audit.exposed_schemas')::text[])`
rather than `= 'public'`. The queries in this document are written with `public` inline for
readability — **substitute the resolved set** when you run them, and say in the report which
schemas were covered. `public` is a default, not a finding.

### What this audit assumes

**Clients reach the database only through PostgREST, Storage, Realtime, and Edge
Functions** — i.e. the Supabase proxy — and the only roles they can act as are `anon`,
`authenticated`, and whatever the JWT names. Every check below is written against that
boundary.

That assumption is usually true on managed Supabase and often **false** on self-hosted or
BYO-Postgres deployments. Check 4e tests it rather than trusting it. If it doesn't hold —
port 5432 reachable from the internet, extra login roles, a pooler in transaction mode —
say so at the top of the report, because a green audit under a false assumption is worse
than no audit.

### Read-only discipline

This audit reads catalogs and grants. Check 7 writes, but only rows it creates itself,
inside a transaction it rolls back. **Never `SELECT` real tenant rows to "prove" a
finding** — a policy definition plus a grant plus a reachable code path is sufficient
proof, and pulling another tenant's data into a transcript *is* the breach you're
reporting.

---

## Severity

| Icon | Level | Meaning |
|------|-------|---------|
| ❌ | **error** | A tenant can reach another tenant's data, or write into it. Live leak. |
| ⚠️ | **warning** | Reachable under a non-trivial condition, or an isolation control not enforced where it's claimed. |
| ℹ️ | **info** | Hygiene, surface reduction, convention. |
| 🟢 | **clear** | Verified correct — record it so the next auditor doesn't re-derive it. |
| ⚪ | **skipped** | No live connection, or not applicable to this project. |

**Bound the panic.** A finding plus a verified-clean perimeter is far more useful than a
finding alone. Say what you checked and found safe, not only what's broken.

---

## Check 1: advisor_baseline

Cheapest pass, run it first. Supabase ships a security linter; don't hand-derive what it
already knows. Dashboard → Advisors → Security, `supabase db advisors --linked --type security`,
or `get_advisors(type: "security")` via the Supabase MCP server. Not `supabase inspect db` —
that reports bloat/index/query stats, not advisor lints.

Lints that matter here: `rls_disabled_in_public`, `policy_exists_rls_disabled`,
`rls_enabled_no_policy`, `security_definer_view`, `function_search_path_mutable`,
`auth_users_exposed`, `extension_in_public`.

### ❌ Error-level
- Any `rls_disabled_in_public` or `auth_users_exposed` finding.

### ⚠️ Warning-level
- Any other open security-category advisor finding.

### ℹ️ Info-level
- **A clean advisor run is not a pass.** The advisors check *structure* — is RLS on, is the
  view a definer view. They cannot check *tenant correctness*: a table with RLS enabled and
  a policy of `USING (true)` is advisor-clean and wide open. Record "advisors clean" and
  keep going. A report that stops here is not an audit.

---

## Check 2: rls_coverage

Which tables are actually protected on the deployed system.

```sql
select n.nspname                                            as schema,
       c.relname                                            as table,
       c.relrowsecurity                                     as rls_enabled,
       c.relforcerowsecurity                                as rls_forced,
       (select count(*) from pg_policy p where p.polrelid = c.oid) as policies
from pg_class c
join pg_namespace n on n.oid = c.relnamespace
where c.relkind in ('r','p')
  and n.nspname not in ('pg_catalog','information_schema','pg_toast')
order by c.relrowsecurity, c.relforcerowsecurity, n.nspname, c.relname;
```

### ❌ Error-level
- **`rls_enabled = false` on any table reachable by `anon`/`authenticated`.** Cross-check
  grants from Check 4 — a table with no RLS *and* no grant is not a leak, and calling it one
  buries the real findings.
- **`rls_forced = false` on a table whose owner is also used at runtime.** RLS is bypassed
  for the table **owner** and for any `BYPASSRLS` role, so a `SECURITY DEFINER` function
  owned by the table owner reads it with no policies applied. `ALTER TABLE … FORCE ROW LEVEL
  SECURITY` closes that and costs nothing on tables the app only reaches through
  `anon`/`authenticated`.

**You cannot apply that rule without knowing who owns what**, so enumerate the ownership
graph rather than assuming everything is `postgres`:

```sql
select pg_get_userbyid(c.relowner) as owner, count(*) as tables,
       count(*) filter (where not c.relforcerowsecurity) as not_forced,
       string_agg(c.relname, ', ' order by c.relname)    as which
from pg_class c join pg_namespace n on n.oid = c.relnamespace
where c.relkind in ('r','p') and n.nspname = 'public'
group by 1 order by 2 desc;
```

A tenant-scoped table owned by anything other than the expected admin role is ❌ until
explained: ownership is a silent RLS bypass for that role, it does not appear in any grant
table, and a migration run under an unusual role is enough to create one. Pair the owner
list with the login-role list from Check 4e — an owner that is also a **login** role is the
combination that turns a schema-hygiene note into a reachable bypass.

### ⚠️ Warning-level
- **`rls_enabled = true, policies = 0`** — deny-all. Not a leak today, but it means nobody
  has reasoned about that table, and the fix applied under deadline pressure is
  `USING (true)`. Flag each one with its grant state.
- Any table **new since the last audit**. The leak is nearly always the table added last
  month.

### ℹ️ Info-level
- Tables in non-exposed schemas with RLS off — note them, and *verify* the schema is
  unexposed in Check 5 rather than assuming it.

---

## Check 3: policy_shape

The policies themselves, read from `pg_policies` rather than from the migration files — the
only version that accounts for drift, partially-applied migrations, and hand-edits.

```sql
select tablename, policyname, cmd, roles, permissive, qual, with_check
from pg_policies
where schemaname = 'public'          -- repeat per exposed schema
order by tablename, cmd, policyname;
```

### ❌ Error-level
- **`qual = 'true'` on SELECT for a tenant-scoped table** — the anon key is a full read of
  that table.
- **A write policy whose *effective* check doesn't constrain the tenant column.** Compute the
  effective write predicate as `coalesce(with_check, qual)` — **not** `with_check IS NULL`:

  ```sql
  select tablename, policyname, cmd,
         coalesce(with_check, qual) as effective_write_check,
         (with_check is null) as reuses_using
  from pg_policies
  where schemaname = 'public' and cmd in ('INSERT','UPDATE','ALL')
  order by tablename, cmd;
  ```

  For `UPDATE` and `ALL`, omitting `WITH CHECK` is **not** a hole on its own: PostgreSQL
  reuses the `USING` expression as the check on the new row. So an UPDATE policy of
  `USING (tenant_id = current_tenant())` with no `WITH CHECK` already rejects a cross-tenant
  move. `INSERT` is the case with no `USING` to fall back on.

  The real leak is a `USING` expression that is a fine *read* filter but a weak *write*
  filter, silently promoted into that role. `USING (user_id = auth.uid())` on a row that also
  carries `tenant_id` reuses cleanly and still lets you rewrite `tenant_id` — you remain the
  owner, so the reused check passes. Read every `effective_write_check` and ask one question:
  **does this expression pin the tenant column?** If not, it's ❌ regardless of which clause
  it came from.

  *(Corrected 2026-08-11 — an earlier revision of this skill flagged `with_check IS NULL`
  outright, which over-reports every correctly-written `UPDATE` policy and misses the
  reused-but-too-weak `USING`. The gate rule below changed with it.)*
- **Two policies on the same table + command where one is broad.** RLS is **permissive-OR**:
  a narrow policy does not constrain a broad one sitting beside it. A tightening migration
  written as `DROP POLICY IF EXISTS "<name>"` whose name doesn't match the live policy is a
  **silent no-op** — the DROP succeeds, the broad policy survives, the migration looks
  applied and the leak persists. Count policies per (table, cmd) and read every one.
- **A tenant column defaulted from the session, on a table with no `WITH CHECK`.**
  `DEFAULT auth.uid()` and `DEFAULT (auth.jwt() -> 'app_metadata' ->> 'tenant_id')` appear
  throughout Supabase starter templates and *look* like enforcement. They are not: a column
  default applies only when the column is **omitted** from the `INSERT`. Any client that
  sends the column explicitly overrides it. The default is ergonomics; `WITH CHECK` is the
  control. Enumerate them and pair each against the table's write policy:
  ```sql
  select table_name, column_name, column_default
  from information_schema.columns
  where table_schema = 'public' and column_default ilike '%auth.%'
  order by table_name, column_name;
  ```
  Every hit on a table whose write policy doesn't enforce the same value is a leak.

### ⚠️ Warning-level
- **`cmd = 'ALL'`** — one `USING` expression serves as both the read filter and the write
  filter (`WITH CHECK` defaults to `USING`). Almost never what you want on a tenant table;
  split per operation.
- **`roles = {public}`** — includes `anon`. Name `authenticated` (or a specific role)
  explicitly, so an unauthenticated caller is denied by the *policy*, not merely by a `NULL`
  `auth.uid()` that happens to match nothing.
- **`USING (auth.role() = 'authenticated')` or `USING ((SELECT auth.uid()) IS NOT NULL)`**
  on a tenant table — logged-in is not authorized. Any user of *any* tenant satisfies it.
- **Column-unscoped UPDATE on a row two tenants can both satisfy** (shared, junction, or
  cross-tenant collaboration rows). RLS scopes **rows, not columns**: if the row predicate
  is satisfiable by the non-owning party and that role holds table-wide `UPDATE`, they can
  rewrite *any* column on that row — including content the other party owns. Only
  column-level `GRANT UPDATE(col, …)` fixes it; verify grant breadth in Check 4.
- **Association and identity tables left open beside a narrowed content table.** Narrowing
  `documents` while `document_shares` and `profiles` sit at `USING (true)` lets any
  authenticated user JOIN them back into "which tenant holds what." Treat association +
  identity tables as part of the same isolation surface as the content they point at.
- **`RESTRICTIVE` vs `PERMISSIVE` confusion.** Restrictive policies AND; permissive ones OR.
  A tenant guard written as permissive *adds* access beside another permissive policy
  instead of removing it. A tenant predicate is often best expressed as a single
  `AS RESTRICTIVE` policy on every command, with permissive policies layering per-feature
  rules on top.
- **No index on the column(s) the policy filters on** — performance, not security, but slow
  policies are what drive someone to "temporarily" widen them later.

### ℹ️ Info-level
- Policy name doesn't follow `<table>_<operation>_<scope>` (e.g. `invoices_select_tenant`).
- `auth.uid()` called bare rather than as `(SELECT auth.uid())` — the subselect form is
  evaluated once per statement instead of once per row.

---

## Check 4: bypass_surfaces

Everything RLS does not cover. **This is where multi-tenant apps actually leak** — the
policies are usually fine and the view over them is not.

### 4a · Views — the biggest blind spot

A Postgres view runs with the privileges of its **owner** unless `security_invoker = true`.
A view owned by `postgres` over an RLS-protected table returns **every tenant's rows** to
whoever can select from it. The base table's policies are never consulted.

```sql
select n.nspname as schema, c.relname as view, c.relkind,
       coalesce((select option_value
                 from pg_options_to_table(c.reloptions)
                 where option_name = 'security_invoker'), 'off') as security_invoker,
       pg_get_userbyid(c.relowner)                           as owner,
       has_table_privilege('anon', c.oid, 'select')          as anon_select,
       has_table_privilege('authenticated', c.oid, 'select') as auth_select
from pg_class c
join pg_namespace n on n.oid = c.relnamespace
where c.relkind in ('v','m')
  and n.nspname not in ('pg_catalog','information_schema')
order by security_invoker, n.nspname, c.relname;
```

#### ❌ Error-level
- A view with `security_invoker = off` over an RLS-protected table, selectable by `anon` or
  `authenticated`. Fix: `ALTER VIEW <v> SET (security_invoker = true);` then re-verify the
  app still gets what it needs — it will now get *less*, which is the point.
- A **materialized view** exposed to `anon`/`authenticated` over tenant data. Matviews do
  **not** support `security_invoker` and RLS never applies to them; the stored result set is
  a flattened cross-tenant snapshot. Revoke it and serve the data another way.

### 4b · SECURITY DEFINER functions

A definer function bypasses RLS by design, and every function with `EXECUTE` granted is a
PostgREST `/rpc/` endpoint. Postgres grants `EXECUTE` to `PUBLIC` by default and `anon` is a
member of `PUBLIC`, so **a new function is anon-callable unless explicitly revoked**.

```sql
select p.proname, p.prosecdef, p.proconfig,
       pg_get_userbyid(p.proowner)                              as owner,
       has_function_privilege('anon', p.oid, 'execute')          as anon_exec,
       has_function_privilege('authenticated', p.oid, 'execute') as auth_exec
from pg_proc p
join pg_namespace n on n.oid = p.pronamespace
where n.nspname = 'public' and p.prokind = 'f'
order by p.prosecdef desc, anon_exec desc, p.proname;
```

Triage by **identity source** — not by whether a `REVOKE` exists. The question is never
"is this function locked down," it is **"whose identity does it act on, and can an arbitrary
caller reach a privileged action."**

| Bucket | Signature | Verdict |
|---|---|---|
<!-- identity-source-buckets v1 — inlined from the canonical taxonomy; re-sync when the marker changes -->

| **1 · param-identity** | takes a tenant/user id as a **parameter** and uses it in `WHERE`/`INSERT`/`UPDATE` with no `caller ∈ tenant` check | ❌ **critical** — pass any tenant id, get any tenant's data. RLS is bypassed, so nothing else stops it. |
| **2 · unguarded privileged write** | definer + write to a sensitive table (roles, credits, subscriptions), no caller authorization, reachable by `authenticated`/`anon` | ❌ **critical** — forgery / privilege escalation |
| **3 · session-identity** | tenant derived **solely** from `(SELECT auth.uid())` or a server-controlled claim, every read and write scoped to it | 🟢 **safe** even if `PUBLIC`-callable — `anon` gets zero rows. Missing `REVOKE` is ℹ️ hygiene. |
| **4 · trigger function** | `RETURNS TRIGGER`, not reachable as `/rpc` | ⚪ on the *call* axis — but see the confused-deputy note below; still needs `search_path` |

Fix for bucket 1: guard with
`IF p_target IS DISTINCT FROM (SELECT auth.uid()) AND coalesce((SELECT auth.role()),'') <> 'service_role' THEN RAISE EXCEPTION 'forbidden'; END IF;`
— or drop the parameter and derive identity internally. Fix for bucket 2: revoke from
`PUBLIC, anon, authenticated` and grant to `service_role` only.

**Do not flood the report.** Missing-`REVOKE` on a bucket-3 function is ℹ️, not ❌. Marking
every un-revoked function critical buries the bucket-1 hole that is the actual breach.

Also ❌ here:
- **`SECURITY DEFINER` with no pinned `search_path`.** Detect it by looking for the entry,
  not by `proconfig IS NULL` — a function with `SET statement_timeout` has a non-null
  `proconfig` and no `search_path` at all:
  ```sql
  select p.proname, p.proconfig
  from pg_proc p join pg_namespace n on n.oid = p.pronamespace
  where n.nspname = 'public' and p.prosecdef
    and not exists (
      select 1 from unnest(coalesce(p.proconfig, '{}')) e where e like 'search_path=%'
    );
  ```
  **Do not blanket-recommend `search_path = pg_catalog, public`.** Two problems with it:
  `public` is writable on many projects (Check 4b's `CREATE` query tells you whether it is
  here), and `pg_temp` is searched **implicitly for tables and views even when unlisted**, so
  a temp table can shadow a name the function relies on. Prefer, in order:
  1. `SET search_path = ''` and schema-qualify every reference in the body. Verbose, and the
     only form with nothing left to resolve.
  2. `SET search_path = pg_catalog, <specific trusted schema>, pg_temp` — naming `pg_temp`
     **last** so it stops being searched first.

  Read the actual entry rather than checking for presence: a pinned path that starts with
  `"$user"`, or lists a schema anyone can create in, is the same hijack with a green tick.
- **`REVOKE … FROM anon` without also revoking `PUBLIC`** — a no-op, since `anon` inherits
  `EXECUTE` through `PUBLIC`. The pattern must be `REVOKE … FROM PUBLIC, anon`.
- **Dynamic SQL** (`EXECUTE format(…)`) with interpolated identifiers or values not passed
  via `USING`, inside a definer function.
- **Session-scoped `set_config` or `SET SESSION` in a function body.** `set_config(name,
  value, false)` — third argument `false`, or a bare `SET`/`RESET` rather than `SET LOCAL` —
  writes a GUC that **outlives the transaction**. Aimed at `request.jwt.claims`, `role`, or
  `search_path` that is a direct identity rewrite: the function edits the value every policy
  in this skill derives the tenant from. It is worse under a pooler (Check 4e), where the
  poisoned setting rides the pooled server connection into whichever tenant is served next.
  ```sql
  select p.proname, p.prosecdef
  from pg_proc p join pg_namespace n on n.oid = p.pronamespace
  where n.nspname = 'public'
    and (pg_get_functiondef(p.oid) ~* 'set_config\s*\([^)]*,\s*false\s*\)'
      or pg_get_functiondef(p.oid) ~* '(^|\s)set\s+(session\s+)?(role|search_path|request\.)');
  ```
  Read every hit: the legitimate form is `set_config(…, true)` / `SET LOCAL`, scoped to the
  transaction. Anything else is ❌ regardless of what the function is for.
⚠️ **Who can create objects in a schema on the search path.** A hijackable `search_path` is
only exploitable if the attacker can *put something there*. On a default Supabase project
`authenticated` has no `CREATE` on `public`, which is what makes the definer rule above a
hardening item rather than a live breach — so verify that assumption rather than inheriting
it, because a project that granted it turns every mutable-`search_path` function into a
reachable one:
```sql
select n.nspname,
       has_schema_privilege('anon', n.oid, 'create')          as anon_create,
       has_schema_privilege('authenticated', n.oid, 'create')  as auth_create
from pg_namespace n
where n.nspname not in ('pg_catalog','information_schema','pg_toast');
```
`true` in either column is ❌ on its own, independent of any function. Note that
`SECURITY INVOKER` functions run as the caller, so a mutable path there is *not* privilege
escalation on its own — it becomes one only via this grant, or when the function is called
from inside a definer context.

ℹ️ **`EXECUTE` granted to `authenticated` on a trigger function.** Revoke it — a trigger
function is invoked by the table event, never by a caller, so the grant buys nothing and
only widens the RPC surface. PostgREST excludes `RETURNS TRIGGER` functions from its schema
cache, so this is hygiene rather than an exposed endpoint; **verify against your own
PostgREST version** before downgrading or escalating it.

**Bucket 4 is not automatically safe — it is only un-*callable*.** "Not reachable as
`/rpc`" answers the wrong question. A `SECURITY DEFINER` trigger on a table users can write
runs with owner privileges on **input the user chose**, which is a confused deputy: B inserts
a row into a table B legitimately owns, the trigger fires as owner, and whatever it does next
— writing an audit row, updating a counter on a parent, fanning out a notification, copying a
field into another table — happens with RLS switched off and a value B supplied.

For each definer trigger, ask what it *writes* and whether any user-supplied column reaches
that write:

```sql
select t.tgname, c.relname as on_table, p.proname, p.prosecdef,
       has_table_privilege('authenticated', c.oid, 'insert') as user_can_insert,
       has_table_privilege('authenticated', c.oid, 'update') as user_can_update
from pg_trigger t
join pg_class c on c.oid = t.tgrelid
join pg_proc  p on p.oid = t.tgfoid
join pg_namespace n on n.oid = c.relnamespace
where not t.tgisinternal and p.prosecdef
  and n.nspname = 'public'
order by c.relname, t.tgname;
```

❌ when a definer trigger on a user-writable table writes to a table the user could not write
directly, using a value from `NEW`. The fix is almost never to drop the trigger — it is to
re-derive the sensitive columns inside it (`auth.uid()`, a membership lookup) instead of
trusting `NEW`.


### 4c · Grants — the drift surface

The migration tree is intent; the grant tables are truth. Grants get hand-edited in the SQL
editor and never make it back into a migration.

```sql
-- table-level reach
select table_schema, table_name, grantee,
       string_agg(privilege_type, ',' order by privilege_type) as privs
from information_schema.role_table_grants
where grantee in ('anon','authenticated','PUBLIC')
  and table_schema not in ('pg_catalog','information_schema')
group by 1,2,3
order by 1,2,3;

-- column-level write breadth on a shared-row table (pairs with Check 3)
select grantee, privilege_type, string_agg(column_name, ', ' order by column_name) as cols
from information_schema.column_privileges
where table_schema = 'public' and table_name = '<table>'
  and grantee in ('anon','authenticated') and privilege_type in ('INSERT','UPDATE')
group by 1,2;
```

#### ❌ Error-level
- `anon` holding any privilege on a table with RLS disabled.
- **Column-`UPDATE` breadth exceeding the app's actual write surface** on a table whose
  UPDATE policy a non-owner can satisfy → content-tamper vector. Fix:
  `REVOKE UPDATE ON <t> FROM authenticated; GRANT UPDATE(<safe cols>) ON <t> TO authenticated;`
- A live grant no migration accounts for — an untracked manual change. Reconcile it into a
  migration or revoke it; either way it must stop being invisible.

#### ❌ Error-level — the two privileges RLS does not govern

`TRUNCATE` and `REFERENCES` are **not** filtered by row-level security. A policy that
perfectly scopes `DELETE` to one tenant does nothing about `TRUNCATE`, which removes every
row in the table for everyone. `REFERENCES` lets the grantee create a foreign key against the
table, which is an existence oracle by construction (see matrix case 13) and can block A's
deletes from B's schema.

```sql
select table_schema, table_name, grantee, privilege_type
from information_schema.role_table_grants
where privilege_type in ('TRUNCATE','REFERENCES')
  and grantee not in ('postgres')            -- plus your platform-owned roles
order by 1,2,3;
```

Check **inherited** grants too — `authenticated` may hold these through a role it is a member
of rather than directly, and `role_table_grants` shows the grantee as named:

```sql
select r.rolname as member, g.rolname as inherits_from
from pg_auth_members m
join pg_roles r on r.oid = m.member
join pg_roles g on g.oid = m.roleid
where r.rolname in ('anon','authenticated','authenticator');
```

Any `TRUNCATE` or `REFERENCES` reaching `anon`/`authenticated`, directly or by inheritance,
is ❌. `GRANT ALL` is the usual source, and it is why `GRANT ALL` should never appear on a
tenant table.

#### ⚠️ Warning-level
- `GRANT … ON ALL TABLES IN SCHEMA public TO anon, authenticated` or
  `ALTER DEFAULT PRIVILEGES … TO anon` anywhere in the tree — **every future table is
  exposed the moment it's created**, which turns Check 2's "new table" warning into a
  standing leak generator.
- `GRANT … TO PUBLIC` — every role inherits it.

### 4d · The service role

The service role carries `BYPASSRLS`. Nothing in this skill protects you from it; the only
controls are that it never reaches a client, and that every server path using it filters by
tenant *in code*.

- Grep the shipped client bundle, mobile app, and any `NEXT_PUBLIC_*` / `VITE_*` /
  `EXPO_PUBLIC_*` env var for **both** key generations. ❌ on any hit — and the key must be
  **rotated**, not just removed, since it lives in every prior build artifact.

  | Generation | Secret (bypasses RLS) | Publishable (safe in a client) |
  |---|---|---|
  | current | `sb_secret_…`, and anything in `SUPABASE_SECRET_KEYS` | `sb_publishable_…` |
  | legacy | `service_role` JWT | `anon` JWT |

  **A `sb_secret_…` key is not a JWT**, so decoding a payload and reading its `role` claim —
  the check an earlier revision of this skill described — silently misses the entire current
  key format. Match on the literal prefixes first, and only decode when the value actually
  looks like a JWT. Grep for `sb_secret_`, `service_role`, and `SUPABASE_SECRET_KEYS`; treat
  `sb_publishable_` and the `anon` JWT as the expected client-side values.
- List every edge function and server route that constructs a service-role client, and
  confirm each derives the tenant from the *verified* session — never from a request body
  field. A service-role handler that trusts `body.tenant_id` is a bucket-1 function wearing
  a different hat.

**Filters that look like authorization.** Separately from key exposure, read every edge
function and server route for the shape `.from(t).select().eq('tenant_id', body.tenantId)`
— a predicate taken from the request rather than the verified JWT. Severity depends
entirely on what is underneath it:

| Client the handler built | Verdict |
|---|---|
| service-role client | ❌ — nothing else is checking; this *is* the authorization, and it's attacker-supplied |
| authenticated client, table has correct RLS | ℹ️ — RLS still holds; the filter only narrows. Note it, don't escalate it. |
| authenticated client, table failed Check 2 or 3 | ❌ — the filter was the only control and it isn't one |

Grade these against the catalog findings rather than on sight. The same line of code is
harmless or critical depending on a fact three checks away, and reporting it without that
join is how an audit loses its reader.

### 4e · The connection boundary — test the assumption, don't inherit it

Everything above assumes callers arrive as `anon`/`authenticated` through the Supabase
proxy. **RLS is enforced per-role, not per-route**, so a direct connection is not itself a
bypass — but it changes *which role* is available, and that is the bypass. On managed
Supabase this check is usually three green lines; on self-hosted it is often where the whole
audit turns.

**Login roles and `BYPASSRLS`.** The highest-value query in this sub-check, and cheap:

```sql
select rolname, rolcanlogin, rolsuper, rolbypassrls, rolreplication
from pg_roles
where rolcanlogin or rolbypassrls or rolsuper
order by rolbypassrls desc, rolsuper desc, rolname;
```

- ❌ **any role with `rolbypassrls` beyond the expected platform set.** `BYPASSRLS` is total
  and unconditional — no policy in this document constrains it. One unexpected row here
  invalidates every other check.
- ❌ a login role outside the managed set (`postgres`, `authenticator`, and the platform's
  own) — especially one an operator created for a script, a BI tool, or a migration runner.
  Those are rarely policy-audited and frequently password-authenticated.
- ⚠️ `rolsuper` on anything the application can authenticate as.

Read `pg_roles`, not `pg_shadow`/`pg_authid` — the former is world-readable and carries what
you need; the latter requires superuser and returns password hashes an audit has no reason
to touch.

**What `authenticator` can reach directly.** PostgREST logs in as `authenticator` and
`SET ROLE`s to `anon`/`authenticated` per request. It should hold **no table privileges of
its own** — only `GRANT anon, authenticated TO authenticator`. Any direct grant is reachable
by anyone who obtains that connection string and simply declines to switch roles:

```sql
select table_schema, table_name,
       string_agg(privilege_type, ',' order by privilege_type) as privs
from information_schema.role_table_grants
where grantee = 'authenticator'
group by 1,2 order by 1,2;
```
❌ on any row.

**Network reachability and pooling.** Confirm whether port 5432 (and the pooler port) is
reachable from outside, and which mode the pooler runs in.

- ⚠️ direct Postgres exposed to the public internet. Not a leak on its own — the roles above
  are what make it one — but it removes the proxy from the trust argument, so every finding
  in Checks 3–6 loses its "only reachable via PostgREST" mitigation.
- ⚠️ **transaction-mode pooling plus any finding from the `set_config` rule in 4b.** In
  transaction mode a server connection is handed to a different client after each
  transaction; a GUC set without `LOCAL` persists on it. That is the one configuration where
  a *session-scoped* identity setting becomes a cross-tenant leak with no policy defect
  anywhere. Session-mode pooling and direct connections don't have this property.

State the outcome in the report even when clean — "connections arrive only via the proxy;
no unexpected login or `BYPASSRLS` roles" is a load-bearing sentence that the rest of the
audit rests on.

---

## Check 5: platform_surfaces

The parts of Supabase that aren't tables, and are routinely audited last or never.

### 5a · Exposed schemas
PostgREST serves the schemas in its `db-schemas` setting (Dashboard → Settings → API) and
nothing else. Confirm the list. A helper schema someone assumed was private (`internal`,
`private`, `app`) that made it onto that list exposes every table in it, subject only to its
grants.

### 5b · Storage
```sql
select id, name, public from storage.buckets;
select policyname, cmd, roles, qual, with_check
from pg_policies where schemaname = 'storage' order by policyname;
```
- ❌ `public = true` on a bucket holding tenant files. A public bucket is an
  unauthenticated CDN read of every object in it; `storage.objects` policies do not apply to
  public reads.
- ❌ Object policies that don't pin the tenant segment of the path — e.g. missing
  `(storage.foldername(name))[1] = <tenant>::text`. Without it, "read your own files" is
  "read any file."
- ⚠️ Long-lived signed URLs. They leak by being forwarded and no policy revokes them before
  expiry. Record the TTL the app uses.

### 5c · Realtime
```sql
select schemaname, tablename from pg_publication_tables where pubname = 'supabase_realtime';
```
Every table in that publication broadcasts its changes. Realtime applies the subscriber's
RLS to `postgres_changes`, but the failure modes are specific enough to test rather than
assume: a table added to the publication *before* its policies existed, and broadcast /
presence channels whose authorization is a separate control from table RLS. ⚠️ any published
table that Check 2 flagged, and confirm with a real second-tenant socket in Check 7.

**Broadcast and presence are a different control, and this is the gap most audits leave
open.** `postgres_changes` is the only Realtime feature RLS reaches. Broadcast and presence
carry no table behind them — a collaborative app pushing cursors, typing indicators, live
document edits, or notification fan-out over broadcast gets **no isolation from anything
above**. Authorization for those channels is RLS on `realtime.messages` (private channels)
plus whatever the client sets; a channel that is not private is joinable by any client that
knows its name, and channel names are usually a predictable function of a tenant or
document id.

```sql
select policyname, cmd, roles, qual, with_check
from pg_policies where schemaname = 'realtime' order by tablename, policyname;
```

- ❌ App code that calls `.channel(name)` without `{ config: { private: true } }` for any
  channel carrying tenant data, or a `realtime.messages` table with no policies while
  private channels are in use.
- ⚠️ Channel names derived from a guessable id. Enumerable names plus a non-private channel
  is a cross-tenant subscription with no database involvement at all.

Test it in Check 7: B joins the channel name A is using and asserts silence.

### 5d · The `auth` schema
- ❌ Any view, function, or foreign-key-driven embed that surfaces `auth.users` rows —
  emails across tenants is the classic. Mirror only what you need into a `profiles` table
  under its own policy.

---

## Check 6: claim_provenance

**Where does the tenant identifier come from?** No advisor lint covers this, and the broken
version is one word different from the correct one — it reviews clean.

Supabase JWTs carry both `user_metadata` and `app_metadata`. **`user_metadata` is
user-writable**: any authenticated user can set arbitrary values in it via
`supabase.auth.updateUser({ data: { … } })`, and they appear in their next JWT. A policy
that reads the tenant from there lets any user mint themselves membership of any tenant.

```sql
-- policies keyed on a user-writable claim
select tablename, policyname, cmd, qual, with_check
from pg_policies
where qual ilike '%user_metadata%' or with_check ilike '%user_metadata%';

-- functions that read or write the user-writable side
select p.proname
from pg_proc p join pg_namespace n on n.oid = p.pronamespace
where n.nspname in ('public','auth')
  and pg_get_functiondef(p.oid) ilike '%raw_user_meta_data%';
```

Classify every tenant predicate in the system by its source:

| Source | Trust | Note |
|---|---|---|
| `auth.uid()` → membership-table lookup | 🟢 **trusted** | Server-controlled at both ends. The default correct answer. |
| `auth.jwt() -> 'app_metadata' ->> 'tenant_id'` | 🟢 **trusted** | Only settable with the service role — but verify *nothing client-reachable writes it*. An edge function that copies a request field into `app_metadata` re-opens the hole. |
| `auth.jwt() -> 'user_metadata' ->> 'tenant_id'` | ❌ **untrusted** | User-writable. Critical. |
| a custom claim from an auth hook | ⚠️ → verify | Trusted only if the hook derives it server-side from a table and cannot be influenced by sign-up input. Inspect it — see below. |
| a function or RPC **parameter** | ❌ **untrusted** | Bucket 1 from Check 4b. |
| a request header or body field | ❌ **untrusted** | |

### Verifying a custom-access-token hook

A hook claim is the one row in that table you cannot grade by reading the policy — the
verdict lives in the hook body. Do not mark it ⚠️ and move on; resolve it to 🟢 or ❌:

```sql
select p.proname, p.prosecdef, p.proconfig,
       has_function_privilege('authenticated', p.oid, 'execute') as auth_exec,
       pg_get_functiondef(p.oid)                                 as body
from pg_proc p join pg_namespace n on n.oid = p.pronamespace
where p.proname ilike '%access_token%' or p.proname ilike '%custom_claim%';
```

Read the body for four things:

1. **Where the tenant comes from.** It must be `select … from <membership table> where
   user_id = (event->>'user_id')::uuid`. If it reads `raw_user_meta_data` and copies it into
   the claim, the hook is **laundering a user-writable value into a trusted-looking one** —
   ❌, and strictly worse than reading `user_metadata` in the policy directly, because every
   downstream policy now looks correct.
2. **Whether anything writes `raw_app_meta_data` from input.** Grep the whole schema, not
   just the hook — a sign-up trigger that seeds `app_metadata` from `raw_user_meta_data` puts
   the user in control of the trusted side. ❌.
3. **`EXECUTE` on the hook.** It should be granted to `supabase_auth_admin` and revoked from
   `authenticated`, `anon`, and `PUBLIC`. A caller-invokable hook is a bucket-1 function.
4. **`prosecdef` and `proconfig`.** Definer with a pinned `search_path`, per Check 4b.

### ❌ Error-level
- Any tenant predicate resolving to a user-writable source.
- **A membership table whose own policies let a user insert their own membership row.** The
  lookup is only as trustworthy as the table behind it, and this is the most common way an
  otherwise-correct pattern is defeated. Membership writes must be service-role-only, or
  gated by an invite flow the invitee cannot self-approve.
- **A recursion workaround that bypasses what it was working around.** A policy that queries
  its own table recurses, and the standard fix is a `SECURITY DEFINER` helper such as
  `user_is_member_of(tenant)`. That helper runs without RLS — if it takes *both* the tenant
  and the user as parameters and merely compares them to each other, it will confirm any
  membership it is asked about. It must derive the user from `auth.uid()` internally.

### ⚠️ Warning-level
- A nullable tenant column under a policy of `tenant_id = current_tenant()`. `NULL = x` is
  `NULL`, not `true`, so orphan rows are invisible rather than leaked — but a policy written
  with `IS NOT DISTINCT FROM`, or a `COALESCE` fallback to a default tenant, pools those
  rows into somebody's account. Check the nullability and the default together.
- Tenant resolved in application code and passed *down* into the query, where the database
  could have derived it from the session. Works until one call site forgets.

---

## Check 7: tenant_b_matrix

Everything above is static analysis: it reads what the system *says*. This check makes the
system **demonstrate** isolation — two real tenants, a full CRUD matrix, no service role.

**Nothing else in this skill replaces it.** A report without it should say so plainly rather
than implying the project was proven safe.

Full harness, SQL and REST forms, and the complete case list: **[`tenant-matrix.md`](tenant-matrix.md)**,
beside this file. The cases that must appear:

| # | Case | Expected |
|---|---|---|
| 1 | B `SELECT`s each table | 0 rows of A |
| 2 | B `INSERT`s a row carrying **A's** tenant id | rejected |
| 3 | B `UPDATE`s one of B's own rows, setting tenant id to **A's** | `42501` specifically — `UPDATE 0` is **inconclusive**, not a pass |
| 4 | B `UPDATE`s / `DELETE`s a row of A's by primary key | 0 rows affected |
| 5 | B reads each **view** and calls each **`/rpc/` function** | no A data |
| 6 | B requests a PostgREST **embed** (`?select=*,child(*)`), nested two deep | no A data |
| 7 | B requests `count=exact` filtered to A's rows | 0, not a positive count |
| 8 | B `INSERT`s a value colliding with A's on a unique constraint | must not confirm A's row exists |
| 9 | B subscribes to realtime on each published table; A writes | B receives nothing |
| 10 | B requests a storage object under A's path prefix | denied |
| 11 | B upserts (`ON CONFLICT DO UPDATE`) onto a key held by A | must not mutate A's row, and must not confirm it exists |
| 12 | B joins **and sends on** the broadcast/presence channel A uses | join denied; send rejected, confirmed from A's side |
| 13 | B inserts a child row referencing **A's** parent | rejected — and `23503` indistinguishable from a random UUID |
| 14 | unauthenticated · anonymous sign-in · second B user · viewer vs admin · removed member · **removed member's unexpired JWT** | 0 rows of A each; the stale-JWT row is the one that fails |
| 15 | Edge Functions, server routes, GraphQL, exports, admin/impersonation routes, called as B with A's ids | no A data |

### ❌ Error-level
- Any returned row, accepted write, or received event in cases 1–6, 9, 10, 12.
- **Case 11 — an upsert that mutates A's row.** RLS *does* apply to the `DO UPDATE` path, so
  this should be impossible; if it happens, the table's UPDATE policy is satisfiable by B and
  Check 3 missed it.

### ⚠️ Warning-level
- **Case 7 — a count that reveals rows the caller cannot read.** An existence oracle.
- **Case 8 — a `23505` unique-violation whose error text names A's row.** An enumeration
  oracle: it confirms which emails, slugs, or domains exist in other tenants. Catch it and
  return a generic conflict.
- **Case 11 — an upsert that errors *differently* depending on whether A holds the key.**
  The same oracle as case 8 by another route, and the one people miss because the upsert
  looks like a write test rather than a read test. `DO NOTHING` is the mirror image: it
  swallows the conflict silently, so B's insert vanishes with a success status and no row —
  not a leak, but a data-loss bug worth reporting in the same breath.
- **Case 3 returning `UPDATE 0`** — the row predicate excluded B's own row, so `WITH CHECK`
  was never reached and the case proved nothing. Fix the fixture and re-run; do not score it
  as a pass.
- Any distinguishable difference between "does not exist" and "exists, denied" — status
  code, latency, or message. Both must look identical from outside.

### ⚪ Skipped
- Report `⚪ tenant_b_matrix: not run` and state plainly that isolation was **not
  demonstrated**, only inspected.

---

## Check 8: regression_gate

The leak that matters is the table added next month. A one-time audit decays; a gate does
not.

**Build it as an allowlist diff, not a zero-count assertion.** A gate that asserts "zero
rows match this bad pattern" works exactly until the first legitimate exception — a
reporting view that really is meant to be `security_invoker = off`, a genuinely public
lookup table — at which point the gate goes permanently red and somebody deletes it. Then
you have neither the exception documented nor the gate. An allowlist survives that: the
exception gets written down, reviewed once, and everything *else* still fails loudly.

The shape: one checked-in file per rule holding the expected set, and the gate diffs live
state against it. Anything in live that isn't in the file fails; anything in the file that
isn't live is stale and warns.

```
.rls-gate/
  rls-disabled.allow        # tables intentionally without RLS (each line: table  # why, who, when)
  definer-views.allow       # views intentionally running as owner
  anon-grants.allow         # every grant anon is meant to hold
  definer-functions.allow   # SECURITY DEFINER functions, with their bucket
  bypassrls-roles.allow     # roles allowed to hold BYPASSRLS
```

```bash
# per rule: live set minus allowlist must be empty
psql "$DATABASE_URL" -Atc "$QUERY" | sort > /tmp/live.txt
grep -vE '^\s*(#|$)' .rls-gate/$RULE.allow | awk '{print $1}' | sort > /tmp/allow.txt
comm -23 /tmp/live.txt /tmp/allow.txt | tee /tmp/new.txt
[ ! -s /tmp/new.txt ] || { echo "unreviewed $RULE:"; cat /tmp/new.txt; exit 1; }
comm -13 /tmp/live.txt /tmp/allow.txt | sed 's/^/stale allowlist entry: /'   # warn only
```

Rules worth gating, each one query feeding the diff above:

| Rule | Live set |
|---|---|
| `rls-disabled` | tables in exposed schemas with `relrowsecurity = false` |
| `weak-write-check` | policies on `INSERT`/`UPDATE`/`ALL` whose `coalesce(with_check, qual)` does not reference the tenant column |
| `definer-views` | views/matviews in exposed schemas without `security_invoker = true` |
| `mutable-search-path` | `SECURITY DEFINER` functions with no `search_path=` entry in `proconfig` |
| `anon-grants` | every table and `EXECUTE` grant held by `anon` |
| `bypassrls-roles` | roles with `rolbypassrls` (Check 4e) |

Two properties this buys that a count never does: an exception carries a **reason and an
author** in the file, so review has something to argue with; and the diff names *what* is
new rather than reporting that a number went up, so the failure is actionable from the CI
log alone.

Seed the allowlists from this audit's own output — every finding you *accept* as
intentional becomes a line, and every finding you fix becomes an absence.

### ⚠️ Warning-level
- No gate exists, and the project has more than one engineer or more than one tenant.
- A gate exists but is **count-based** — it will be deleted at the first legitimate
  exception, so it is a gate with an expiry date.

### ℹ️ Info-level
- Allowlist entries with no reason, no author, or no date.
- Allowlist entries that no longer match anything live (stale — the object was dropped, and
  the exception should go with it).

---

## Output Format

```
RLS AUDIT — <project / schema>            <linked ref or "static">
═══════════════════════════════════════════════════
Check 1: advisor_baseline .............. ✅ | ⚠️ N | ❌ N
Check 2: rls_coverage .................. ✅ | ⚠️ N | ❌ N
Check 3: policy_shape .................. ✅ | ⚠️ N | ❌ N
Check 4: bypass_surfaces ............... ✅ | ⚠️ N | ❌ N
  ↳ views / definer fns / grants / service-role / connection boundary
Check 5: platform_surfaces ............. ✅ | ⚠️ N | ❌ N
  ↳ schemas / storage / realtime / auth
Check 6: claim_provenance .............. ✅ | ⚠️ N | ❌ N
Check 7: tenant_b_matrix ............... ✅ | ❌ N | ⚪ skipped
Check 8: regression_gate ............... ✅ | ⚠️ N

───────────────────────────────────────────────────
FINDINGS  (reachable-first)
❌ [check] <canonical harm in one line>
   <where> — <mechanism> — <who can reach it>
   Fix: <the statement to run>
...
───────────────────────────────────────────────────
VERIFIED CLEAR
🟢 <surface> — <why it holds>
───────────────────────────────────────────────────
CONNECTION BOUNDARY: proxy-only, no unexpected login/BYPASSRLS roles | <what differs>
ISOLATION DEMONSTRATED: yes (matrix 15/15) | no (static only)
```

Both trailer lines are load-bearing and neither is optional. The first states the assumption
the other seven checks rest on; the second states whether isolation was proven or merely
inspected. A report that omits them is asserting more than it verified.

Lead with the **canonical harm** — "any authenticated user can read every tenant's invoices
via `v_invoice_summary`" — then the mechanism, then the fix. Not "view lacks
security_invoker."

---

## Principles

- **Advisors check structure; you check tenancy.** `USING (true)` is advisor-clean.
- **The leak is never in the table you're thinking about.** It's in the view over it, the
  RPC that reads it, the matview that cached it, or the bucket the files went to.
- **Read is half the surface.** A missing `WITH CHECK` is the leak nobody tests for, because
  testing it means writing.
- **Trace the tenant id to its source.** A correct-looking policy over a user-writable claim
  is worse than no policy, because it reviews clean.
- **Static analysis inspects; the matrix demonstrates.** Never report isolation as verified
  on inspection alone.
- **Bound the blast radius.** Say what you checked and found clean. A finding without a
  perimeter becomes a week of undirected panic.
- **Postgres behavior you can derive; platform behavior you must observe.** Catalog semantics
  are stable and safe to reason about — `coalesce(with_check, qual)`, owner bypass, FK checks
  running below RLS. Realtime, Storage, PostgREST and the auth layer are versioned products
  whose behavior has changed and will change again, and reasoning "Postgres does X, therefore
  Supabase exposes X" is how a confident wrong claim gets into an audit. Every platform-level
  assertion in this skill either carries a verify-this note or is a bug. Capture the actual
  payload, status, or error and record what you saw, with the version.
- **Don't prove it with real data.** Grants plus policy plus a reachable path is proof
  enough.
