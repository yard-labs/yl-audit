# Scan patterns

Use Grep with these patterns. Adjust paths to the repo (`src/`, `app/`, `pages/`, `lib/`). Exclude `node_modules`. Never print matched secret values.

## Inventory
- `package.json`: `@supabase/supabase-js`, `@supabase/ssr`, `prisma`, `@prisma/client`, `drizzle-orm`, `drizzle-kit`, `kysely`, `typeorm`, `pg`, `postgres`.
- Config and CI: `.github/workflows/*`, `supabase/config.toml`, `supabase/migrations/`, `drizzle/`, `prisma/migrations/`, `vercel.json`, `.env*.example`.

## Browser reach (client code)
- Find client files: `'use client'` / `"use client"` (Next.js), anything under `src/` for Vite or CRA, `components/`, `hooks/`.
- Client creation: `createClient(`, `createBrowserClient(`, `createClientComponentClient(`.
- Public env use: `NEXT_PUBLIC_SUPABASE`, `VITE_SUPABASE`, `REACT_APP_SUPABASE`, `import.meta.env.VITE_`.
- Data calls: `.from(`, `.rpc(`, `.storage.from(` inside client files. `supabase.auth.*` alone is Auth-only use.
- Layout preconnect or config that names the project URL is not a connection by itself.

## Anon in play
- Anon key variables: `SUPABASE_ANON_KEY`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `sb_publishable_`.
- SQL: `to anon`, `for select using (true)`, `grant .* to anon`, `auth.role() = 'anon'`, policies without a `to` clause (apply to public).
- Pages or routes that query without a session check.

## Key leakage
- `service_role`, `SUPABASE_SERVICE_ROLE_KEY`, `sb_secret_`, `SERVICE_ROLE` in files that are client code, in `NEXT_PUBLIC_`/`VITE_` variables, or in built output (`.next/static`, `dist`, `build`).
- JWT shapes: `eyJ[A-Za-z0-9_-]{10,}\.eyJ` then decode only the `role` claim of the payload to see anon vs service_role.
- Connection strings: `postgres(ql)?://[^ ]+:[^ @]+@`, `DATABASE_URL`, `DIRECT_URL`, `pooler.supabase.com`, `db\.[a-z0-9]+\.supabase\.co`.
- Tracked env files: `git ls-files | grep -E '(^|/)\.env'`.
- History (read-only): `git log --all -S'SERVICE_ROLE' --oneline`, `git log --all --diff-filter=D --name-only | grep -i env`.

## RLS quality (SQL migrations)
- Per table: `create table` vs `enable row level security` / `force row level security`.
- Weak predicates: `using (true)`, `with check (true)`, `auth.role() = 'authenticated'`, `auth.uid() is not null`, domain-only checks on `auth.jwt() ->> 'email'`, any use of `user_metadata`.
- Good signs: policies calling a permission or role function, `(select auth.uid())`, per-role `to` clauses.
- Views: `create view` without `security_invoker`. Functions: `security definer` without `set search_path`.
- Grants and defaults: `grant `, `revoke `, `alter default privileges`. Storage: policies on `storage.objects`.
- Leftover copies: schemas or tables named `backup`, `_old`, dated snapshots.

## MFA
- App code: `auth.mfa`, `mfa.enroll`, `mfa.challenge`, `mfa.verify`, `getAuthenticatorAssuranceLevel`, `aal2`, `totp`. None of these in a sign-in flow means the app does not use MFA.
- SQL: `auth.jwt() ->> 'aal'`, `aal2` in policies on sensitive tables.
- Report no MFA on sign-in as a finding (Access control) where the app holds sensitive data. The dashboard and hosting accounts are a separate check (see the settings table).

## Server side
- Service key in routes: `createClient(` with a service key variable, `SUPABASE_SERVICE_ROLE_KEY` use.
- Caller gates: the project's auth helper (for example `requireUser`) at the top of every route handler. List routes without one.
- ORM connection: pooler port `6543` (transaction mode) vs `5432`; `pgbouncer=true`; `prepare: false`; `directUrl` for migrations.
- SSL: `sslmode=verify-full`, `ssl: { ca`, versus `rejectUnauthorized: false`, `sslmode=disable`, `NODE_TLS_REJECT_UNAUTHORIZED=0`.

## Delivery
- CI migrate step: `drizzle-kit migrate`, `prisma migrate deploy`, `supabase db push`, `kysely migrate`, `typeorm migration:run`, and whether a deploy job `needs:` it.
- Manual signals: docs saying "applied by hand", scripts named `migrate` run from a laptop, migrations that exist only in the dashboard.
- Tests or checks asserting RLS is on for every table; a drift check job.
