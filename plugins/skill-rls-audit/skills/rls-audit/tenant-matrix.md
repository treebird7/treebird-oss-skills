# The two-tenant matrix — harness and cases

> Companion to `/rls-audit` Check 7. Everything else in that skill *inspects* the system;
> this makes it *demonstrate* isolation. Run it against staging or a preview branch with
> production-shaped schema — never against production with real tenant data in the blast
> radius.

## The rule

**Never use the service role.** It carries `BYPASSRLS`, so every case passes and the run
proves nothing. Tenant B acts with the **anon key plus B's own access token**, exactly as
the browser would. If a case can only be made to pass by escalating the credential, the case
failed.

---

## Setup

Provision two tenants that look nothing alike, so a leak is unmistakable in the output:

- **Tenant A** — `Acme`, one user `a@example.test`, seeded with at least one row in *every*
  tenant-scoped table, each carrying an obvious marker (`'AAAA-CANARY'`).
- **Tenant B** — `Beta`, one user `b@example.test`, seeded the same way with `'BBBB-CANARY'`.

Seed through the app's own write path where possible. Seeding *with* the service role is
fine — only the **assertions** must run unprivileged.

Then grep every response body in the whole run for `AAAA-CANARY` while acting as B. One
match anywhere is a failed audit, whatever the individual case assertions said.

---

## Form 1 — SQL (fastest; covers cases 1–4, 8)

Runs against the database directly, impersonating a session. Use it to sweep every table in
minutes. It does **not** exercise PostgREST, so it cannot cover embeds, counts, or the API's
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

-- ── case 2: write INTO tenant A
insert into public.invoices (tenant_id, amount) values ('<TENANT_A_ID>', 1);
-- expect: ERROR 42501 new row violates row-level security policy
-- a successful INSERT here is the missing-WITH_CHECK leak

-- ── case 3: move a row OUT of B into A
update public.invoices set tenant_id = '<TENANT_A_ID>' where marker = 'BBBB-CANARY';
-- expect: ERROR 42501 — NOT "UPDATE 0"

-- ── case 4: reach A's row by primary key
update public.invoices set amount = 999 where id = '<A_ROW_ID>';     -- expect UPDATE 0
delete from public.invoices              where id = '<A_ROW_ID>';    -- expect DELETE 0

rollback;                 -- resets role and claims with the transaction
```

Two failure modes that look like passes:

- **Running as a superuser without `set local role`.** `postgres` has `BYPASSRLS`, so every
  case "passes" by cheerfully returning A's rows. That is why `select current_user` is in
  the harness — check it, don't assume it.
- **Running as the table owner.** RLS is skipped for the owner unless the table has
  `FORCE ROW LEVEL SECURITY`. `authenticated` is not the owner, which is the entire point of
  the role switch.

Note the asymmetry between cases 3 and 4: a **rejected** write (`42501`) and a **zero-row**
write are both safe, but they mean different things. `UPDATE 0` in case 3 means the row
predicate excluded it; an *error* means `WITH CHECK` caught it. Case 3 returning `UPDATE 1`
is the leak.

### Sweeping every table

Generate the case-1 probe for the whole schema rather than hand-writing it:

```sql
select format('select %L as tbl, count(*) from public.%I;', c.relname, c.relname)
from pg_class c
join pg_namespace n on n.oid = c.relnamespace
where c.relkind in ('r','p','v','m') and n.nspname = 'public'
order by c.relname;
```

Run the generated statements inside the impersonation transaction above. Every count must be
0 for A's canary **and greater than 0 for B's own rows — assert both.** A harness that only
checks "0 of A's rows" passes trivially against a broken connection, an empty database, or a
policy that denies everything including B.

---

## Form 2 — REST (covers cases 5–8, 10)

This is the surface an attacker actually has. Get two real tokens using the **anon** key:

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

**Decode `$BTOK` before you start** — `jq -R 'split(".")[1]|@base64d|fromjson' <<<"$BTOK"` —
and confirm its `role` is `authenticated`, not `service_role`. Then:

```bash
# ── case 5a: every view
as_b "v_invoice_summary?select=*"                    # expect none of A's rows

# ── case 5b: every RPC, called with A's identifiers
as_b "rpc/get_invoice" -X POST -H 'Content-Type: application/json' \
     -d '{"p_invoice_id":"<A_ROW_ID>"}'              # expect error or null

# and with A's tenant id, if the function takes one — this IS the bucket-1 test
as_b "rpc/list_invoices" -X POST -H 'Content-Type: application/json' \
     -d '{"p_tenant_id":"<TENANT_A_ID>"}'            # expect error or []

# ── case 6: embeds, two deep
as_b "invoices?select=*,line_items(*),customer:customers(*,contacts(*))"

# ── case 7: count oracle
as_b "invoices?select=id&tenant_id=eq.<TENANT_A_ID>" -H 'Prefer: count=exact' -I
# read Content-Range: expect 0/*  — a positive count with an empty body IS the oracle

# ── case 8: enumeration via unique violation
as_b "customers" -X POST -H 'Content-Type: application/json' \
     -d '{"email":"<AN_EXISTING_TENANT_A_CUSTOMER_EMAIL>"}'
# a 409 whose message quotes the constraint and the value confirms A's row exists
```

### Case 10 — storage

```bash
# as tenant B
curl -s -o /dev/null -w '%{http_code}\n' \
  "$URL/storage/v1/object/tenant-files/<TENANT_A_ID>/secret.pdf" \
  -H "apikey: $ANON" -H "Authorization: Bearer $BTOK"        # expect 400/403, not 200

# unauthenticated, to catch a public bucket
curl -s -o /dev/null -w '%{http_code}\n' \
  "$URL/storage/v1/object/public/tenant-files/<TENANT_A_ID>/secret.pdf"
```

Also **list**, don't only fetch — `POST /storage/v1/object/list/<bucket>` with `prefix: ""`.
Object *names* leak tenant structure (customer names, invoice numbers) even when the bytes
themselves are protected.

---

## Case 9 — realtime

For each table in the `supabase_realtime` publication: subscribe as B, write to that table
as A, and assert B receives nothing.

```js
// as tenant B — anon key + B's access token
const b = createClient(URL, ANON, {
  global: { headers: { Authorization: `Bearer ${BTOK}` } },
})
await b.realtime.setAuth(BTOK)          // required — the socket authenticates separately

const seen = []
b.channel('probe')
 .on('postgres_changes', { event: '*', schema: 'public', table: 'invoices' },
     p => seen.push(p))
 .subscribe()

// … now, as tenant A, insert/update a row in public.invoices …
// assert seen.length === 0
```

Two things silently invalidate this case: forgetting `setAuth`, which leaves the socket on
the anon token and can make it look correctly restrictive for entirely the wrong reason; and
asserting too early — wait until at least one *B-owned* write has arrived, so you know the
subscription is live before concluding that silence means isolation.

---

## Reporting

For each case record **case · surface · expected · observed · verdict**, then one summary
line the audit report carries verbatim:

```
ISOLATION DEMONSTRATED: 10/10 cases across 14 tables, 3 views, 6 RPCs, 2 buckets, 4 published tables
```

Partial runs must say which cases were skipped. "9/10, case 9 skipped (no realtime in this
project)" is a fine result. "Isolation verified" over a run that never subscribed to
anything is not.

---

## Extending it

When an audit finds a leak by hand that the matrix missed, **add the case here before fixing
the bug.** That is the compounding mechanism: the matrix becomes the accumulated memory of
every leak you have ever found, and a case costs one line to run forever.

Candidates worth adding as you meet them: soft-deleted rows still visible to the other
tenant; `ON CONFLICT DO UPDATE` reaching a row the caller cannot `SELECT`; full-text search
and `tsvector` columns indexing across tenants; aggregate or reporting endpoints that return
per-row detail; CSV and PDF exports built server-side with a privileged client; webhook and
email templates rendered with cross-tenant context; and any admin route that accepts a
tenant id as a parameter.
