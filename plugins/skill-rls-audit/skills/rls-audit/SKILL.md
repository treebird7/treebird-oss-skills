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
already knows. Dashboard → Advisors → Security, or `get_advisors(type: "security")` via the
Supabase MCP server.

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
- **A write command with `with_check IS NULL`** (`cmd` in `INSERT`, `UPDATE`, `ALL`). The
  single highest-yield finding in this skill: read is scoped, write is not. The caller can
  `INSERT` rows carrying another tenant's id, and on `UPDATE` can move an existing row *out
  of* their tenant and into another. **A read-only test suite never sees it**, which is why
  it survives so long in real projects.
- **Two policies on the same table + command where one is broad.** RLS is **permissive-OR**:
  a narrow policy does not constrain a broad one sitting beside it. A tightening migration
  written as `DROP POLICY IF EXISTS "<name>"` whose name doesn't match the live policy is a
  **silent no-op** — the DROP succeeds, the broad policy survives, the migration looks
  applied and the leak persists. Count policies per (table, cmd) and read every one.

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
| **1 · param-identity** | takes a tenant/user id as a **parameter** and uses it in `WHERE`/`INSERT`/`UPDATE` with no `caller ∈ tenant` check | ❌ **critical** — pass any tenant id, get any tenant's data. RLS is bypassed, so nothing else stops it. |
| **2 · unguarded privileged write** | definer + write to a sensitive table (roles, credits, subscriptions), no caller authorization, reachable by `authenticated`/`anon` | ❌ **critical** — forgery / privilege escalation |
| **3 · session-identity** | tenant derived **solely** from `(SELECT auth.uid())` or a server-controlled claim, every read and write scoped to it | 🟢 **safe** even if `PUBLIC`-callable — `anon` gets zero rows. Missing `REVOKE` is ℹ️ hygiene. |
| **4 · trigger function** | `RETURNS TRIGGER`, not reachable as `/rpc` | ⚪ n/a on this axis; still needs `search_path` |

Fix for bucket 1: guard with
`IF p_target IS DISTINCT FROM (SELECT auth.uid()) AND coalesce((SELECT auth.role()),'') <> 'service_role' THEN RAISE EXCEPTION 'forbidden'; END IF;`
— or drop the parameter and derive identity internally. Fix for bucket 2: revoke from
`PUBLIC, anon, authenticated` and grant to `service_role` only.

**Do not flood the report.** Missing-`REVOKE` on a bucket-3 function is ℹ️, not ❌. Marking
every un-revoked function critical buries the bucket-1 hole that is the actual breach.

Also ❌ here:
- **`SECURITY DEFINER` with no `SET search_path`** (`proconfig IS NULL`) — search-path
  hijack. Needs `SET search_path = pg_catalog, public`, with `pg_catalog` first.
- **`REVOKE … FROM anon` without also revoking `PUBLIC`** — a no-op, since `anon` inherits
  `EXECUTE` through `PUBLIC`. The pattern must be `REVOKE … FROM PUBLIC, anon`.
- **Dynamic SQL** (`EXECUTE format(…)`) with interpolated identifiers or values not passed
  via `USING`, inside a definer function.

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

- Grep the shipped client bundle, mobile app, and every `NEXT_PUBLIC_*` / `VITE_*` /
  `EXPO_PUBLIC_*` variable for a service-role key — decode the JWT's payload segment and
  read its `role` claim rather than eyeballing the prefix. ❌ if found, and the key must be
  **rotated**, not merely removed: it is in every prior build artifact.
- List every edge function and server route that constructs a service-role client, and
  confirm each derives the tenant from the *verified* session — never from a request body
  field. A service-role handler that trusts `body.tenant_id` is a bucket-1 function wearing
  a different hat.

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
| a custom claim from an auth hook | ⚠️ | Trusted only if the hook derives it server-side from a table and cannot be influenced by sign-up input. |
| a function or RPC **parameter** | ❌ **untrusted** | Bucket 1 from Check 4b. |
| a request header or body field | ❌ **untrusted** | |

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
| 3 | B `UPDATE`s one of B's own rows, setting tenant id to **A's** | rejected |
| 4 | B `UPDATE`s / `DELETE`s a row of A's by primary key | 0 rows affected |
| 5 | B reads each **view** and calls each **`/rpc/` function** | no A data |
| 6 | B requests a PostgREST **embed** (`?select=*,child(*)`), nested two deep | no A data |
| 7 | B requests `count=exact` filtered to A's rows | 0, not a positive count |
| 8 | B `INSERT`s a value colliding with A's on a unique constraint | must not confirm A's row exists |
| 9 | B subscribes to realtime on each published table; A writes | B receives nothing |
| 10 | B requests a storage object under A's path prefix | denied |

### ❌ Error-level
- Any returned row, accepted write, or received event in cases 1–6, 9, 10.

### ⚠️ Warning-level
- **Case 7 — a count that reveals rows the caller cannot read.** An existence oracle.
- **Case 8 — a `23505` unique-violation whose error text names A's row.** An enumeration
  oracle: it confirms which emails, slugs, or domains exist in other tenants. Catch it and
  return a generic conflict.
- Any distinguishable difference between "does not exist" and "exists, denied" — status
  code, latency, or message. Both must look identical from outside.

### ⚪ Skipped
- Report `⚪ tenant_b_matrix: not run` and state plainly that isolation was **not
  demonstrated**, only inspected.

---

## Check 8: regression_gate

The leak that matters is the table added next month. A one-time audit decays; a gate does
not.

Recommend — and offer to write — a CI check that fails on:

1. Any table in an exposed schema with `relrowsecurity = false`.
2. Any policy on `INSERT`/`UPDATE`/`ALL` with `with_check IS NULL`.
3. Any view or matview in an exposed schema without `security_invoker = true`.
4. Any `SECURITY DEFINER` function with `proconfig IS NULL`.
5. Any grant to `anon` not present in a checked-in allowlist.

Each is one catalog query whose row count must be zero, so the gate is a short SQL file plus
a `--csv | test -z` wrapper — cheap enough that "we'll add it later" isn't a real argument.

### ⚠️ Warning-level
- No gate exists, on a project with more than one engineer or more than one tenant.

### ℹ️ Info-level
- A gate exists but its allowlist hasn't been reviewed since the last audit.

---

## Output Format

```
RLS AUDIT — <project / schema>            <linked ref or "static">
═══════════════════════════════════════════════════
Check 1: advisor_baseline .............. ✅ | ⚠️ N | ❌ N
Check 2: rls_coverage .................. ✅ | ⚠️ N | ❌ N
Check 3: policy_shape .................. ✅ | ⚠️ N | ❌ N
Check 4: bypass_surfaces ............... ✅ | ⚠️ N | ❌ N
  ↳ views / definer fns / grants / service-role
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
ISOLATION DEMONSTRATED: yes (matrix 10/10) | no (static only)
```

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
- **Don't prove it with real data.** Grants plus policy plus a reachable path is proof
  enough.
