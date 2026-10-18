# Settings checks (for the user to run)

The code scan cannot see project settings or live database state. Give the user these checks, with the expected result for each. **Do not run them yourself** (no network, no tokens). All of them are read-only.

Notes for the user:
- Replace `<ref>` with the project reference. Keep any access token in an environment variable (`SUPABASE_ACCESS_TOKEN`, from the dashboard's account tokens page), never in a chat, a report or the repo.
- Run SQL in the dashboard SQL editor, or with `psql` over the pooler using a verified SSL connection.
- Command and endpoint names change. Confirm against the current Supabase documentation before relying on them.
- Never paste keys, passwords or connection strings back into the audit. Counts, flags and names are enough.

## Project settings

| Setting | Dashboard path | Read-only check | Expected |
|---|---|---|---|
| Plan, backups, point-in-time recovery | Project Settings > Billing; Database > Backups | Dashboard only | Paid plan, daily backups listed, PITR if wanted |
| Log retention | Logs | Dashboard only | Retention that matches the plan and incident needs |
| Compute size and pausing | Project Settings > Compute and Disk | Dashboard only | Size fits the largest tables; a paid project does not pause when idle |
| Enforce SSL | Database > Settings > SSL Configuration | `supabase ssl-enforcement get --project-ref <ref>` or `GET https://api.supabase.com/v1/projects/<ref>/ssl-enforcement` with a bearer token | Enforcement on for database connections |
| Network restrictions | Database > Settings > Network Restrictions | `supabase network-restrictions get --project-ref <ref>` | Only the backend, CI and office addresses, if used |
| Data API exposure | Project Settings > API (Data API settings) | `GET https://api.supabase.com/v1/projects/<ref>/postgrest`, or the SQL below | Only the schemas the browser needs; none if the Data API is off |
| API keys | Project Settings > API Keys | `supabase projects api-keys --project-ref <ref>` (prints keys: do not paste output) | Legacy keys disabled or rotated if ever exposed; secret keys only on the server; a list of who holds them |
| Sign-up and providers | Authentication > Sign In / Providers | `curl -s -H "apikey: <anon key>" https://<ref>.supabase.co/auth/v1/settings` (public settings) | Email sign-up off unless needed, only intended providers, auto-confirm off |
| MFA for people with dashboard access | Organization settings (security / MFA) and each member's account | Dashboard only; ask the owner to list who has access and whether MFA is on for each (check the current docs for organisation-wide enforcement and which plan includes it) | Every owner, admin and developer account has MFA, enforced organisation-wide where the plan allows |
| MFA for the app's own users | Authentication > Sign In / Providers (multi-factor) and Authentication > Users | `GET https://api.supabase.com/v1/projects/<ref>/config/auth` with a bearer token, or the dashboard; for enforcement see the SQL below | Multi-factor enabled, and the app requires it at sign-in (or policies on sensitive tables require the `aal2` assurance level) for staff and admins |
| Site URL, redirect URLs, JWT expiry, password protection, SMTP, rate limits | Authentication > URL Configuration, Sessions, Attack Protection | `GET https://api.supabase.com/v1/projects/<ref>/config/auth` with a bearer token | No wildcard redirects; sensible expiry; protections on |

## Database state (SQL)

```sql
-- Session is on SSL (shows your own connection, not proof of enforcement)
select ssl, version from pg_stat_ssl where pid = pg_backend_pid();

-- Schemas the Data API exposes, and role settings
select current_setting('pgrst.db_schemas', true);
select rolname, rolconfig from pg_roles where rolname in ('authenticator','anon','authenticated');

-- Tables without row-level security
select n.nspname, c.relname from pg_class c join pg_namespace n on n.oid=c.relnamespace
where c.relkind in ('r','p') and not c.relrowsecurity
  and n.nspname not in ('pg_catalog','information_schema','pg_toast');

-- All policies (look for true, auth.role(), roles {public}, unexpected names)
select schemaname, tablename, policyname, roles, cmd, qual, with_check from pg_policies order by 1,2,3;

-- Grants to anon, authenticated and PUBLIC
select table_schema, table_name, grantee, privilege_type
from information_schema.role_table_grants
where grantee in ('anon','authenticated','PUBLIC') and table_schema not in ('pg_catalog','information_schema');

-- Default privileges for every owner
select pg_get_userbyid(defaclrole) owner, defaclnamespace::regnamespace, defaclobjtype, defaclacl from pg_default_acl;

-- Views that are not security_invoker
select n.nspname, c.relname, c.reloptions from pg_class c join pg_namespace n on n.oid=c.relnamespace
where c.relkind in ('v','m') and n.nspname not in ('pg_catalog','information_schema')
  and not coalesce(c.reloptions @> array['security_invoker=true'], false);

-- Security-definer functions and their search_path
select n.nspname, p.proname, p.proconfig, p.proacl from pg_proc p join pg_namespace n on n.oid=p.pronamespace
where p.prosecdef and n.nspname not in ('pg_catalog','information_schema');

-- MFA enforced in policies (aal2), and how many users have enrolled a factor (counts only)
select count(*) as policies_requiring_aal2 from pg_policies where qual ilike '%aal2%' or with_check ilike '%aal2%';
select count(distinct user_id) as users_with_a_factor from auth.mfa_factors where status = 'verified';

-- Storage buckets and policies
select id, name, public from storage.buckets;
select policyname, roles, cmd, qual from pg_policies where schemaname='storage';

-- Extensions, scheduled jobs (do not copy command text), vault secret names only
select extname from pg_extension order by 1;
select jobid, schedule, jobname, active from cron.job;
select name, created_at from vault.secrets;

-- Optional, for internal tools on one email domain: users outside it, and unconfirmed users (counts only)
select count(*) filter (where lower(email) !~ '^[^@]+@(example\.com)$') as outside_domain,
       count(*) filter (where email_confirmed_at is null) as unconfirmed
from auth.users;
```

Adjust the domain in the last query, or skip it for customer-facing apps.

## Anon exposure (when the Data API is on)

The anon key is public by design, so the question is what the `anon` role can reach. Expected result for every query: no rows, or only objects the product deliberately makes public.

```sql
-- Tables and views anon can read or write
select c.oid::regclass as relation,
       has_table_privilege('anon', c.oid, 'select') as anon_select,
       has_table_privilege('anon', c.oid, 'insert') as anon_insert,
       has_table_privilege('anon', c.oid, 'update') as anon_update,
       has_table_privilege('anon', c.oid, 'delete') as anon_delete
from pg_class c join pg_namespace n on n.oid=c.relnamespace
where c.relkind in ('r','p','v','m') and n.nspname not in ('pg_catalog','information_schema','pg_toast')
  and (has_table_privilege('anon', c.oid, 'select') or has_table_privilege('anon', c.oid, 'insert')
    or has_table_privilege('anon', c.oid, 'update') or has_table_privilege('anon', c.oid, 'delete'));

-- Functions anon can call (default EXECUTE goes to PUBLIC unless revoked)
select n.nspname, p.proname, p.prosecdef as security_definer
from pg_proc p join pg_namespace n on n.oid=p.pronamespace
where n.nspname not in ('pg_catalog','information_schema','pg_toast','extensions','graphql','graphql_public','realtime','vault','net','pgsodium','storage','supabase_functions')
  and has_function_privilege('anon', p.oid, 'execute');

-- Schemas anon can use
select nspname from pg_namespace where nspname not like 'pg_%' and has_schema_privilege('anon', oid, 'usage');

-- Policies that apply to anon or to everyone
select schemaname, tablename, policyname, roles, cmd, qual from pg_policies
where roles @> '{anon}' or roles @> '{public}';

-- Public storage buckets
select id, name, public from storage.buckets where public;
```

Also run the outside-in test below with the anon key against every table and RPC the product uses.

## Outside-in test (read-only)

With the project's public anon key (public by design, still keep it out of reports), request an empty result from each table:

```
curl -s -o /dev/null -w '%{http_code}\n' -H "apikey: <anon key>" -H "Authorization: Bearer <anon key>" "https://<ref>.supabase.co/rest/v1/<table>?select=id&limit=0"
```

Expected: 401, 403 or a permission-denied error for every table. A 200 with `[]` means anon can reach the table; row-level security may still hide the rows, so flag it to verify. Never run write requests.

## Writing the "To verify" line

Every unverified item in the report uses this form, filled in from the tables above:

> **To verify:** open Database > Settings > SSL Configuration in the Supabase dashboard (or run `supabase ssl-enforcement get --project-ref <ref>`). **Expect:** enforcement on for database connections. **If not:** switch it on, then confirm every client connects with certificate verification.

Rules: name the place (dashboard path, SQL editor or command), name the result to expect, and say what a different result means. If only the project owner can see it, say who to ask and what to ask for.

## How to use the results

If the user pastes results (with keys and passwords removed), compare each to the Expected column, mark the setting pass, fail or unverified, and update the level and findings. A setting nobody has checked stays **unverified**, never assumed.
