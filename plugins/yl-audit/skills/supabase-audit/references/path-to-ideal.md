# Path up

Give only the steps from the level found. Each step says what it buys. Prefer the cheapest step that removes the most risk. For small teams, defer later steps until sensitivity or headcount grows, and say so as a pragmatic suggestion, not security advice.

## Applies at any level
Require MFA: on every dashboard, hosting and code-host account that can reach the data, and on the app's own sign-in for staff and admins (Supabase Auth MFA, with sensitive policies requiring `aal2` once everyone has enrolled). Cheap, and it removes the commonest route in, a stolen password. Check the current Supabase docs for which plan includes each option.

## From level 0 (exposed)
1. Rotate every exposed key and password today. Remove it from client code and public env variables.
2. Remove committed secrets from the repo and, with the owner's agreement, from history. Add a secret scan to CI.
3. Restrict third-party keys (for example by API and address).
Buys: stops the immediate compromise. Then continue from level 1.

## From level 1 (open)
1. Enable RLS on every table and remove open policies. Revoke `anon` grants. Set views to `security_invoker`. Add `set search_path = ''` to security-definer functions. On a backend, gate every route.
2. Close default privileges so new tables grant nothing until policies exist.
3. Add a CI check that fails when a table has RLS off.
Buys: level 2 in days.

## From level 2 (flat) to level 4
Choose a route. Both end equally secure.
- **Route A (stay on the Data API):** add a roles and permissions table and per-role or ownership policies, the most sensitive tables first (financial, personal or special-category data). Move write logic into checked functions. Pull sensitive columns out of `select *` reads. Usually the smaller change.
- **Route B (add a backend):** follow "Route B setup" below. More work up front; recommended when portability matters or when logic is outgrowing policies.

## From level 3 (backend, service key) to level 4
1. Create a dedicated least-privilege role and move the request path to it through the pooler. Keep the service key for scheduled and admin jobs only.
2. Enforce and verify SSL. Disable the Data API for the data schemas.
3. Keep a caller gate on every route and a test that proves it by calling handlers, not by searching for a keyword.

## From level 4 to level 5
Put delivery in place (see "Migrations in code, run in CI before deploy"): migrations in the repo, applied by CI before deploy with a separate credential, and a CI check that fails on RLS off or missing grants.

## Route B setup
1. Add a backend (for example Next.js route handlers) that verifies the caller and owns the data calls. Move reads and writes route by route, sensitive tables first.
2. Create a dedicated Postgres role per service with grants only on what it needs. Connect through the pooler.
3. Enforce SSL and verify certificates in the shared connection helper.
4. Switch the Data API off, or stop exposing the data schemas, once nothing in the browser needs it.
5. Recommended: adopt an ORM (below) so data access no longer depends on `supabase-js`.

## ORM guidance (portability, recommended not required)
- **Drizzle**: SQL-first, TypeScript, good fit for Next.js and serverless. `drizzle-kit generate` then `drizzle-kit migrate` in CI. With transaction-mode pooling use `prepare: false`.
- **Prisma**: mature migrations (`prisma migrate deploy` in CI). Use the pooled URL for the app (`pgbouncer=true`) and a direct or session URL (`directUrl`) for migrations.
- **Kysely**: type-safe query builder, thin, easy to move between hosts.
- **TypeORM**: acceptable if the team already uses it.
- Keep `supabase-js` for Auth and Storage only. Put data access behind one module so the host can change by changing a connection string. Check current driver and pooler docs before relying on the settings above.

## Migrations in code, run in CI before deploy
1. All schema, RLS policies, grants and functions live in the repo as migration files. Nothing is changed in the dashboard.
2. A CI workflow on merge to main: a `migrate` job (using a migration credential held as an environment secret, with required reviewers for production) runs the migrate command, and the `deploy` job `needs: migrate`.
3. A pull-request job runs the migrations against a disposable database (a Supabase branch or a service container) and runs RLS and grant checks.
4. Destructive changes use expand then contract (add, migrate readers and writers, then remove).
5. A scheduled drift check compares the repo with the live schema.
6. The runtime credential is a different, narrower role from the migration credential.
