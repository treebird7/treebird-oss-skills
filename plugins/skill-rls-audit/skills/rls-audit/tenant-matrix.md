# The two-tenant matrix — harness and cases

> Companion to `/rls-audit` Check 7. Everything else in that skill *inspects* the system;
> this makes it *demonstrate* isolation. Run it against staging or a preview branch with
> production-shaped schema — never against production with real tenant data in the blast
> radius.

## The rule

**Never use the service role.** It carries `BYPASSRLS`, so every case passes and the run
proves nothing. Tenant B acts with the **anon key plus B's own access token**, exactly as
the browser would. If a case can only be made to pass by escalating the credential, the
case failed.

---

## Setup

Provision two tenants that look nothing alike, so a leak is unmistakable in the output:

- **Tenant A** — `Acme`, one user `a@example.test`, seeded with at least one row in *every*
  tenant-scoped table, each carrying an obvious marker (`'AAAA-CANARY'`).
- **Tenant B** — `Beta`, one user `b@example.test`, seeded the same way with `'BBBB-CANARY'`.

Seed through the app's own write path where possible. Seeding with the service role is fine
— only the *assertions* must run unprivileged.

Then grep every response body in the whole run for `AAAA-CANARY` while acting as B. One
match anywhere is a failed audit, whatever the individual case assertions said.

---

## Form 1 — SQL (fastest; covers cases 1–4, 11)

Runs against the database directly, impersonating a session. Use it to sweep every table in
minutes; it does **not** exercise PostgREST, so it cannot cover embeds, counts, or the API's
error surface — those need Form 2.

```sql
begin;

-- Claims FIRST, then the role: auth.uid() / auth.jwt() read this GUC.
select set_config(
  'request.jwt.claims',
  json_build_object(
    'sub',          '<TENANT_B_USER_UUID>',
    'role',         'authenticated',
    'email',        'b@example.test',
    'app_metadata', json_build_object('tenant_id', '<TENANT_B_ID>')
  )::text,
  true                    -- is_local: dies with the transaction
);
set local role authenticated;

select current_user;      -- MUST be 'authenticated' before any result below is meaningful

-- ── case 1: read
select count(*) from public.invoices where marker = 'AAAA-CANARY';   -- expect 0
select count(*) from public.invoices where marker = 'BBBB-CANARY';   -- expect > 0 (control)

-- ── case 2: write INTO tenant A
savepoint c2;
insert into public.invoices (tenant_id, amount) values ('<TENANT_A_ID>', 1);
-- expect: ERROR 42501 new row violates row-level security policy
rollback to savepoint c2;   -- MANDATORY: the error above aborts the tx without this

-- ── case 3: move a row OUT of B into A
savepoint c3;
update public.invoices set tenant_id = '<TENANT_A_ID>' where marker = 'BBBB-CANARY';
-- expect: ERROR 42501 — NOT "UPDATE 0", NOT "UPDATE 1"
rollback to savepoint c3;

-- ── case 4: reach A's row by primary key
savepoint c4;
update public.invoices set amount = 999 where id = '<A_ROW_ID>';     -- expect UPDATE 0
delete from public.invoices              where id = '<A_ROW_ID>';    -- expect DELETE 0
rollback to savepoint c4;

-- ── case 11: upsert onto a key A holds
savepoint c11;
insert into public.customers (tenant_id, email, name)
values ('<TENANT_B_ID>', '<AN_EXISTING_TENANT_A_CUSTOMER_EMAIL>', 'overwritten')
on conflict (email) do update set name = excluded.name;
-- expect: A's row unchanged. RLS applies to the DO UPDATE path, so a successful
-- mutation here means the UPDATE policy is satisfiable by B and check 3 missed it.
rollback to savepoint c11;
-- then repeat with an email NO tenant holds; if the two responses differ,
-- the upsert is an existence oracle (same class as case 8).

rollback;                 -- resets role and claims with the transaction
```

**Every failing case needs its own savepoint, and this is not stylistic.** In PostgreSQL an
error aborts the whole transaction: after case 2 raises `42501`, every following statement
returns `25P02 current transaction is aborted` until you roll back. Without the savepoints,
cases 3, 4 and 11 never execute — and a runner that only greps for "no rows returned"
reports them as **passing**. That is the worst failure mode this harness can have: it
silently stops testing and reports success. If you prefer, run each case in its own
transaction instead; what you cannot do is run them sequentially in one.


Two failure modes that look like passes:

- **Running as a superuser without `set local role`.** `postgres` has `BYPASSRLS`, so every
  case "passes" by cheerfully returning A's rows. That is why `select current_user` is in
  the harness — check it, don't assume it.
- **Running as the table owner.** RLS is skipped for the owner unless the table has
  `FORCE ROW LEVEL SECURITY`. `authenticated` is not the owner, which is why the role
  switch is the whole harness.

**Case 3 must return `42501` — `UPDATE 0` is a failed test, not a pass.** The two look alike
in a results table and mean opposite things. B *owns* the row it is trying to move, so the
`USING` predicate must match it and `WITH CHECK` must then reject the new tenant id. If you
get `UPDATE 0`, `USING` excluded B's own row — the fixture is wrong (marker mismatch, or B
can't see its own data) and `WITH CHECK` was never exercised at all. Scoring that as a pass
is how a table with **no** `WITH CHECK` gets a clean bill of health from this harness. Fix
the fixture and re-run.

Case 4 is the opposite: there, `UPDATE 0`/`DELETE 0` **is** the pass, because B does not own
the row and `USING` is supposed to exclude it. Same status, opposite meaning, one case apart
— which is exactly why the expected value is written per case rather than as a blanket
"write is rejected."

### Sweeping every table

Generate the case-1 probe for the whole schema rather than hand-writing it — **with the
canary predicate baked in**. A bare `count(*)` counts B's own rows too, so it can neither
show "zero of A's" nor serve as a control; it produces a number that means nothing.

```sql
select format(
         'select %L as tbl, '
         'count(*) filter (where %I = %L) as a_leaked, '   -- must be 0
         'count(*) filter (where %I = %L) as b_control '   -- must be > 0
         'from %I.%I;',
         c.relname,
         'marker', 'AAAA-CANARY',
         'marker', 'BBBB-CANARY',
         n.nspname, c.relname)
from pg_class c
join pg_namespace n on n.oid = c.relnamespace
join pg_attribute a on a.attrelid = c.oid and a.attname = 'marker' and not a.attisdropped
where c.relkind in ('r','p','v','m')
  and n.nspname = any (current_setting('rls_audit.exposed_schemas')::text[])
order by n.nspname, c.relname;
```

Tables with no `marker` column drop out of that join — deliberately. Seed the marker on
every tenant-scoped table during setup, or substitute the tenant column and compare tenant
ids instead (`count(*) filter (where tenant_id = '<TENANT_A_ID>')`). What you must not do is
fall back to a bare count and eyeball it.

**Both columns are assertions.** `a_leaked = 0` alone passes trivially against a broken
connection, an empty database, a typo'd marker, or a policy that denies everything including
B. `b_control > 0` is what proves the probe could have seen something. A run where every
table reports `0 / 0` is not a clean audit — it is a harness that isn't connected to
anything, and it will report a perfect score.

---

## Form 2 — REST (covers cases 5–8, 10)

This is the surface an attacker actually has. Get two real tokens with the **anon** key:

```bash
export URL="https://<ref>.supabase.co"
export ANON="<anon key>"

tok() {                       # tok <email> <password>
  curl -s "$URL/auth/v1/token?grant_type=password" \
    -H "apikey: $ANON" -H "Content-Type: application/json" \
    -d "{\"email\":\"$1\",\"password\":\"$2\"}" | jq -r .access_token
}
export BTOK=$(tok b@example.test '<pw>')

as_b() {                      # as_b <path-and-query> [curl args…]
  curl -s -D /tmp/rls-hdrs "$URL/rest/v1/$1" \
    -H "apikey: $ANON" -H "Authorization: Bearer $BTOK" "${@:2}"
}
```

**Decode `$BTOK` before you start** (`jq -R 'split(".")[1]|@base64d|fromjson'`) and confirm
its `role` is `authenticated`, not `service_role`. Then:

```bash
# ── case 5a: every view
as_b "v_invoice_summary?select=*"                    # expect [] of A's rows

# ── case 5b: every RPC, called with A's identifiers
as_b "rpc/get_invoice" -X POST -H 'Content-Type: application/json' \
     -d '{"p_invoice_id":"<A_ROW_ID>"}'              # expect error or null
# and with A's tenant id, if the function takes one — that IS the bucket-1 test
as_b "rpc/list_invoices" -X POST -H 'Content-Type: application/json' \
     -d '{"p_tenant_id":"<TENANT_A_ID>"}'            # expect error or []

# ── case 6: embeds, two deep
as_b "invoices?select=*,line_items(*),customer:customers(*,contacts(*))"

# ── case 7: count oracle
as_b "invoices?select=id&tenant_id=eq.<TENANT_A_ID>" -H 'Prefer: count=exact' -I
# read Content-Range: expect 0/*  — a positive count with an empty body is the oracle

# ── case 8: enumeration via unique violation
as_b "customers" -X POST -H 'Content-Type: application/json' \
     -d '{"email":"<AN_EXISTING_TENANT_A_CUSTOMER_EMAIL>"}'
# a 409 whose message quotes the constraint and the value confirms A's row exists
```

### Case 10 — storage (read **and** every mutation)

```bash
st() { curl -s -o /dev/null -w '%{http_code}' \
         -H "apikey: $ANON" -H "Authorization: Bearer $BTOK" "$@"; }

FORBIDDEN="tenant-files/<TENANT_A_ID>/secret.pdf"    # exists, belongs to A
ABSENT="tenant-files/<TENANT_A_ID>/no-such-file.pdf" # does not exist

st "$URL/storage/v1/object/$FORBIDDEN"   # → F
st "$URL/storage/v1/object/$ABSENT"      # → N
```

**Do not assert a specific code.** Supabase returns 400 or 404 across versions and paths,
and for both "absent" and "unauthorized." The two assertions that hold regardless:

1. **F is not 2xx.** Any success is the leak.
2. **F == N**, in status *and* body. If a forbidden object answers differently from a
   nonexistent one, storage is an existence oracle — B can enumerate A's filenames without
   reading a byte, and filenames carry customer names and invoice numbers.

Then the unauthenticated probe, which catches a public bucket regardless of any policy:

```bash
curl -s -o /dev/null -w '%{http_code}\n' "$URL/storage/v1/object/public/$FORBIDDEN"
```

**Reads are the half everyone tests.** Run every mutation too — each is a separate policy on
`storage.objects`, and a project that pinned the tenant prefix on `SELECT` has usually not
pinned it on `INSERT`, `UPDATE`, or `DELETE`:

| Op | Probe as B against A's prefix | Expect |
|---|---|---|
| upload | `POST /object/tenant-files/<A>/x.pdf` | denied |
| overwrite | `PUT /object/tenant-files/<A>/secret.pdf` | denied — the quiet one; it destroys A's data without ever reading it |
| upsert | `POST` with `x-upsert: true` | denied |
| copy | `POST /object/copy` with A's path as source | denied |
| move | `POST /object/move` with A's path as source | denied |
| delete | `DELETE /object/tenant-files/<A>/secret.pdf` | denied |
| list | `POST /object/list/tenant-files` with `{"prefix":""}` | none of A's names |
| sign | `POST /object/sign/tenant-files/<A>/secret.pdf` | denied — **a signed URL outlives the policy that should have blocked it** |

The signing row is the one to run first if you run only one: it converts a momentary
authorization mistake into a durable, shareable, un-revokable capability.

---

## Case 9 — realtime

For each table in the `supabase_realtime` publication: subscribe as B, then write to that
table as A, and assert B receives nothing.

```js
// as tenant B — anon key + B's access token
const b = createClient(URL, ANON, { global: { headers: { Authorization: `Bearer ${BTOK}` } } })
await b.realtime.setAuth(BTOK)          // required — the socket authenticates separately
const seen = []
b.channel('probe')
 .on('postgres_changes', { event: '*', schema: 'public', table: 'invoices' },
     p => seen.push(p))
 .subscribe()
// … now, as tenant A, insert/update a row in public.invoices …
// assert seen.length === 0
```

Two things that silently invalidate this case: forgetting `setAuth`, which leaves the
socket on the anon token and can make it look correctly restrictive for the wrong reason;
and asserting too early — wait for at least one *B-owned* write to arrive first, so you
know the subscription is live before concluding that silence means isolation.

**`DELETE` events are a documented hole, not a bug in your policies.** Supabase states that
RLS is **not applied** to Postgres Changes delete events: subscribers receive the deletion
regardless of policy. An INSERT/UPDATE-only probe never surfaces it, which is exactly why it
belongs in the matrix — you are testing for a platform limitation you must design around,
not for a mistake you can fix with a policy.

```js
b.channel('probe-del')
 .on('postgres_changes', { event: 'DELETE', schema: 'public', table: 'invoices' },
     p => seen.push(p))
 .subscribe()
// … as tenant A, delete a row …
// B very likely receives it. That is the platform's behavior.
```

Record the outcome as ⚠️ with the mitigation rather than ❌ against the project: the payload
is limited to the replica identity (primary key by default), so the exposure is "A deleted
row `<uuid>`" unless someone set `REPLICA IDENTITY FULL` — which promotes it to a full
cross-tenant row disclosure and **is** ❌. Check it:

```sql
select pt.schemaname, pt.tablename,
       case c.relreplident
         when 'd' then 'default (pk only)'
         when 'i' then 'index'
         when 'f' then 'FULL — delete events leak the whole row'
         else 'nothing'
       end as replica_identity
from pg_publication_tables pt
join pg_namespace n on n.nspname = pt.schemaname
join pg_class c on c.relname = pt.tablename and c.relnamespace = n.oid
where pt.pubname = 'supabase_realtime'
order by 1, 2;
```

### Case 12 — broadcast and presence

`postgres_changes`, above, is the only Realtime feature RLS reaches. Broadcast and presence
have no table behind them, so **nothing in checks 1–6 constrains them at all** — if the app
pushes cursors, typing indicators, live edits, or notification fan-out over broadcast, this
case is the only thing in the audit that looks at it.

Work out how the app names its channels (usually a template over a tenant, document, or room
id), then have B join the name **A** is using:

```js
const ch = b.channel(`tenant:${TENANT_A_ID}`, { config: { private: true } })
const seen = []
ch.on('broadcast', { event: '*' }, p => seen.push(p))
  .on('presence',  { event: 'sync' }, () => seen.push(ch.presenceState()))
  .subscribe(status => console.log(status))       // expect an auth error, not SUBSCRIBED

// … now, as tenant A, broadcast on the same channel …
// assert seen.length === 0
```

Two verdicts, and the weaker one is the common case: a **subscribe failure** is the pass. If
B reaches `SUBSCRIBED` and merely receives nothing during the window, that is not isolation —
it means nobody sent anything yet. Run A's broadcast *after* B is subscribed and assert on
what arrives.

Also try it **without** `private: true`. A public channel is joinable by anyone who can guess
the name, and channel names are near-always a predictable function of an id the other tenant
can obtain. If the app's own client omits `private`, `realtime.messages` policies never come
into play regardless of how well they're written.

**Joining is only half of it — test *sending*.** Realtime authorizes receive and send
separately (`SELECT` vs `INSERT` policies on `realtime.messages`), so a channel B cannot read
may still be one B can write to. That is not a data leak; it is worse in some apps —
injecting events into A's channel means forged presence, forged notifications, and forged
live edits appearing to A's users as legitimate.

```js
const ch = b.channel(`tenant:${TENANT_A_ID}`, { config: { private: true } })
await ch.subscribe()
const res = await ch.send({ type: 'broadcast', event: 'injected', payload: { x: 1 } })
// expect an error / 'error' status — and confirm from an A-side listener that nothing landed
await ch.track({ user: 'B' })   // presence write — same expectation
```

Assert on **A's side**, not on B's return value. A `send()` that resolves `ok` while nothing
arrives is a client-side illusion; a `send()` that resolves `error` while the event still
lands is the leak. Only A's listener settles it.

---

## Case 13 — cross-tenant foreign keys

Referential integrity is enforced **below** RLS: a foreign-key check consults the parent row
regardless of policy. Two consequences, and both are live in most schemas.

```sql
savepoint c13;
-- B inserts a child row pointing at a parent that belongs to A
insert into public.line_items (invoice_id, tenant_id, amount)
values ('<AN_INVOICE_ID_OWNED_BY_A>', '<TENANT_B_ID>', 1);
rollback to savepoint c13;
```

- **Succeeds** → ❌ relationship corruption: B now owns a row hanging off A's object, and any
  join, aggregate, cascade, or export that walks the relationship crosses the tenant boundary
  without a single policy being violated. The fix is a composite key — `FOREIGN KEY
  (invoice_id, tenant_id) REFERENCES invoices (id, tenant_id)` — so the tenant travels with
  the reference; RLS cannot express this on its own.
- **Fails with `23503`** → still ⚠️ **existence oracle**: the error distinguishes "no such
  parent" from "parent exists but isn't yours." Compare the response against a genuinely
  random UUID. Identical → clean. Different → B can enumerate A's primary keys by watching
  which foreign keys resolve.

Also probe `ON DELETE CASCADE` in the same direction: if B can delete a row whose cascade
reaches A's rows, the cascade executes with no policy check on the far side.

---

## Case 14 — authentication lifecycle

Every case so far runs as one healthy member of tenant B. Isolation also has to hold for the
identities around the edges, and each row here has broken a real application:

| Identity | Probe | Expect |
|---|---|---|
| unauthenticated (anon key, no token) | every case-1 read | 0 rows, no errors that leak schema |
| Supabase **anonymous** sign-in | same | 0 rows — anonymous users are `authenticated`, and policies written as `auth.role() = 'authenticated'` admit them |
| second user **inside** tenant B | case 1 | sees B's rows — the control that proves the policy isn't merely deny-all |
| viewer vs admin within B | privileged RPCs, admin routes | role separation actually enforced, not just rendered |
| **removed** member of tenant A | every read | 0 rows |
| removed member's **old JWT**, not yet expired | every read | **this is the one that fails** |

That last row is the finding. `app_metadata` is stamped into the JWT at issue time, so a
tenant claim carried in the token stays valid until the token expires — revoking membership
in the database changes nothing for up to the full token lifetime. Policies that resolve the
tenant through a **membership-table lookup** keyed on `auth.uid()` revoke instantly; policies
that read `app_metadata.tenant_id` do not.

Test it concretely: capture the token, remove the membership, then replay every read with the
captured token. If rows still come back, record the **token TTL** as the actual revocation
window — that number belongs in the report and in any security questionnaire answer about
offboarding. Shortening the JWT expiry narrows it; only the lookup form closes it.

---

## Case 15 — the application layer

Checks 4d and 5 inspect Edge Functions, server routes, exports, and search endpoints by
reading code. Reading is not demonstrating, and these run with credentials the database
checks cannot see.

For each one, call it as B with A's identifiers in every position that accepts one — path
segment, query string, body field, header:

- Edge Functions and server API routes, especially any that take a tenant, org, or workspace id
- GraphQL (`pg_graphql`), including nested selections and connection `filter` arguments —
  it is a second query planner over the same tables and needs its own probe
- report, analytics, and aggregate endpoints — verify they return **aggregates**, not per-row detail
- CSV / PDF / backup exports, which are usually built server-side with a privileged client
- webhook payloads and rendered email templates, which frequently carry more context than the UI
- any admin or impersonation route ("view as user"), tested as a **non**-admin member of B

Anything reachable here bypasses every catalog query in this skill, because the credential
was chosen by application code rather than derived from B's JWT.


---

## Reporting

For each case record: **case · surface · expected · observed · verdict**. Then one summary
line the audit report carries verbatim:

```
ISOLATION DEMONSTRATED: 15/15 cases across 14 tables, 3 views, 6 RPCs, 2 buckets, 4 published tables
```

Partial runs must say which cases were skipped. "14/15, case 12 skipped (no broadcast in this
project)" is a fine result; "isolation verified" over a run that never subscribed to
anything is not.

---

## Extending it

When an audit finds a leak by hand that the matrix missed, **add the case here before
fixing the bug.** That is the whole compounding mechanism: the matrix is the accumulated
memory of every leak you have ever found, and a case costs one line to run forever.

Candidates worth adding as you meet them: full-text search and `tsvector` columns indexing
across tenants; aggregate or report endpoints returning per-row detail; CSV and PDF exports
built server-side with a privileged client; webhook and email templates rendered with
cross-tenant context; and any admin route that takes a tenant id as a parameter.

**Out of scope on purpose:** soft-deleted rows still readable at the database layer. Real,
and worth fixing — but under a tenant predicate it is *your own* deleted data you can still
read, which is a retention and privacy finding rather than a tenant-isolation one. Only
bring it back here if soft-deletion is what separates two tenants' rows on a shared table,
which is a schema shape you should be arguing against anyway.
