---
name: supabase-audit
description: Audit how an application connects to Supabase by scanning its code (read-only). Flags direct browser connections, anon-key use, leaked keys, and Data API use with and without full row-level security, then levels the result 0 to 5. Two routes are equally secure (Data API with full RLS, or a backend with a least-privilege role); level 5 adds migrations in code applied by CI. Gives ordered steps to move up. Use when reviewing a Supabase app's security architecture, after a client code drop, or before adding features to a Supabase app.
argument-hint: [path to repo, default current directory]
---

# Supabase audit

Scan a codebase and answer three questions: how does it reach the database, is that secure, and what is the next step up. Code scanning only. It cannot prove the live database state, so say what is unverified and list the live checks to confirm.

## Rules

Follow `../app-audit/references/rules.md`. On top of it:
- To learn whether a JWT is an anon or service-role key, print only its `role` claim, never the token.
- Take each To verify line from `references/settings-checks.md` where one exists; otherwise write it.

## What secure looks like (two routes, equally secure)

**Route A: Data API with full RLS.** The browser talks to Supabase with the publishable (anon) key and the user's session. RLS is on every table, with policies keyed to a role, permission or ownership. Views are `security_invoker`, default privileges are closed, security-definer functions check the caller and set `search_path`, storage policies are by role, and `anon` reaches nothing the product does not mean to be public.

**Route B: a backend in front of the data.** The browser never connects to the database. It talks to your own backend (for example Next.js route handlers) that verifies the caller. The backend connects through the Supabase **pooler** as a **dedicated least-privilege Postgres role** (not superuser, not bypass-RLS), with **SSL enforced and certificate verified** (`verify-full`). The Data API is off, or not exposing the data schemas.

**Both routes:** the service-role key exists only in scheduled and admin jobs, never in the browser or the request path; and **migrations live in code and run in CI before deploy**, not by hand and not from the dashboard.

Neither route is a level below the other for security. Route B is still the recommended architecture (see Architecture suggestions in `../code-audit/references/report-template.md`): the database is reached only through a backend, at every product stage. Route B with an **ORM or query builder** (Drizzle, Prisma, Kysely or TypeORM) instead of `supabase-js` for data is the recommended direction when portability matters: Supabase becomes Auth, Storage and Postgres hosting, so moving host is a connection-string change, at about the same security and effort. Present it as a recommendation, never as a finding against a sound Route A app.

## Levels

| Level | Name | What the code shows | Report severity |
|---|---|---|---|
| 0 | Exposed | A service-role key, database password or connection string is reachable from the browser (public env prefix, client file, built bundle) or committed to the repo or its history | High |
| 1 | Open | Data API with RLS missing on some tables, open policies (`using (true)`), `anon` grants or leaking definer views; or a backend with ungated routes | High |
| 2 | Flat | Data API with RLS on everywhere, but one flat rule ("any signed-in user" or "anyone on one email domain") with no per-role or ownership checks | Medium |
| 3 | Backend, service key | Browser never reaches the database. Server routes use the service-role key (bypasses RLS) and every route has a caller gate: one missed gate exposes everything | Low |
| 4 | Secure | Route A or Route B above, in full | None |
| 5 | Secure and delivered | Level 4 plus delivery checks D1, D2, D3 and D7 pass | None |

Where a codebase mixes patterns, report the **lowest** level among the data paths that matter and list the others. Say which route the app is on.

## Delivery checks (scored separately, pass / fail / unknown / n/a)

| # | Check |
|---|---|
| D1 | Schema, policies and grants defined as migrations in the repo (not dashboard-only) |
| D2 | Migrations applied by a CI workflow before deploy (for example a `migrate` job that the `deploy` job `needs`), not by a person |
| D3 | Migration credential held as a CI secret, separate from the app's runtime credential |
| D4 | Route B: an ORM or query builder owns data access (recommended, not required; n/a for Route A) |
| D5 | Route B: pooler connection in the right mode (transaction mode for serverless, direct or session mode for migrations) |
| D6 | Server connections: SSL enforced and verified by the client (no `rejectUnauthorized: false`, no `sslmode=disable` or `require` without a CA) |
| D7 | A CI check fails when a table has RLS off, or an expected policy or grant is missing |
| D8 | Schema drift check between the repo and the live database |

## Procedure

Read `references/scan-patterns.md` for the search patterns, then:

1. **Inventory.** Framework, dependencies (`@supabase/supabase-js`, `@supabase/ssr`, `prisma`, `drizzle-orm`, `kysely`, `pg`, `postgres`, `typeorm`), env example files, workflow files, migration folders.
2. **Browser reach.** Does any client-side code create or import a Supabase client for data, and which key does it use? Distinguish Auth-only use (fine) from data calls (`.from(`, `.rpc(`, `.storage`). This decides the route.
3. **Anon in play.** On Route A the anon key in the client is expected, so judge what `anon` can reach: no table, view, sequence or schema grants, no `to anon` or public-role policies, no `EXECUTE` on functions (Postgres grants it to PUBLIC by default), and no public storage buckets. Give the user the anon checks from `references/settings-checks.md` and mark each unverified until they report back.
4. **Key leakage.** Service-role keys, database URLs or passwords in public env variables, client files, built output (`.next/static`, `dist`), committed `.env` files, and git history. Report location and type only.
5. **RLS quality** (when migrations are present). Per table: RLS on, policy predicates, wildcard or flat rules, role or permission checks, definer views, security-definer functions without `search_path`, default privileges, grants, storage policies, backup schemas.
6. **Server side.** Routes using the service key and whether each has a caller gate. ORM and connection strings: pooler host and port, SSL options, prepared-statement settings.
7. **Delivery.** Walk D1 to D8 using workflow files and migration folders. Treat "applied by hand" in docs as a fail for D2.
8. **Settings.** Read `supabase/config.toml` and similar files for settings held in code (sign-up and confirmation, providers, JWT expiry, exposed schemas, row limit, pooler). For everything else, give the user the checks from `references/settings-checks.md`, each with its expected result. Never run them yourself.

## Output

1. **Verdict:** the level (0 to 5), the route (A, B or mixed), one sentence on why, and the report severity from the table.
2. **Data paths found:** each path (browser direct, server routes, jobs) with its own level and `file:line` evidence.
3. **Findings:** ordered worst first. Each has: title, level it pins the code to, evidence (`file:line`, never a secret), why it matters in one or two sentences, confidence, the smallest next step, and a To verify line if not confirmed.
4. **Delivery checks:** D1 to D8 table.
5. **Path up:** read `references/path-to-ideal.md` and give the ordered steps from the level found, with a rough effort for each (hours, days, weeks) and what each step buys. Say what is cheap now and what to defer until the team or sensitivity grows. Mention the ORM recommendation only as a recommendation.
6. **Settings and how to check them:** a table with these columns: Setting | Status (pass, fail or unverified) | What the code or config shows | **To verify: do this** | Expect. Cover plan and backups, log retention, compute, enforce SSL, network restrictions, Data API exposure, API keys, sign-up and providers, MFA (dashboard accounts and the app's own users), auth options, RLS and grants state, anon exposure, storage, and cron and vault. Offer to interpret the results if the user pastes them (keys and passwords removed).

Keep the tone plain.
