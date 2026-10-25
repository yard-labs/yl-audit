# Report template

## Scorecard
A one-screen summary at the very top of the report. Show a row only for an audit that actually ran; a skipped audit is named "not run", never scored 0.

**Overall risk:** Low / Medium / High / Critical, with one line on why (the worst confirmed finding).

**Stage:** team S1/S2/S3 . product P1/P2/P3 (see `rules.md`).

| Dimension | Level | Right for stage |
|---|---|---|
| Architecture | n / 5 | t |
| Data connection (Supabase) | n / 5 | t |
| CI/CD | n / 5 | t |
| Observability | n / 5 | t |

Give each row a light: green when level >= target, amber one below, red two or more below (or any level-0 condition). Architecture uses the ladder below; the other three come from their own audits. Where Supabase is in use, its connection level feeds architecture's connection pillar, so the two never disagree.

**Findings:** N total: H High, M Medium, L Low. Then the by-category count table from the Categories section (only categories with findings).


## Overview (table: Field | Detail)
What is it | Where it lives | Data present | Access needed | Audit inputs (what was received, which checks, what was not done).

## Architecture overview
A plain-text diagram and short table: components and hosting, data stores and what sensitive data each holds, external integrations, trust boundaries, each data flow with the credential it uses, environments, and how code and migrations reach production. Mark each statement confirmed in code, from docs, or unverified.

## Authentication mechanism
How people sign in, the token or session and where it is held, how the server verifies it, who counts as an authorised user (claims, groups, roles, guests), every other credential in play, session lifetime and revocation, account removal (for internal tools, leaver handling), and anything enforced only in the browser. Include a table of mechanisms and where each is used.

## API endpoint map
The table and summary matrix from `../../code/references/endpoint-map.md`: every endpoint bucketed by auth mechanism, ranked by data risk exposure, with the weak spots listed and tied to findings.

## Findings (table)
ID | Name | Category | Finding | Criticality (High, Medium, Low) | Risk | Remediation

- Plain language for a non-developer reader. One idea per finding. Say "Confirmed in code", "Plausible" or "Unverified". Name file and line only where it helps; never include a secret.
- Order High, then Medium, then Low. IDs stable; do not renumber after sharing.
- Risk says what happens, to whom. Remediation is specific enough to act on ("do X in Y"), and ends "See recommendation N".
- Stage trade-offs: Low, with the pragmatic-suggestion wording, and a note above the table listing them.

## Architecture suggestions
A short, separate section from the findings: how the system should look, given what was found and the team's stage. Suggest, do not demand; scale to the stage (solo and early needs less than a larger team). Cover only what applies:
- **Shape of the system, by product stage** (for an app that reads and writes data, internal or public). Name the stage the app is at and the one it is heading into; suggest only the next step.
  - **Static and read-only** (any stage). A static site with no writes and no per-user data is a sound shape, even for sensitive data, when it sits behind single sign-on or an access gateway. The data is in the build, so everyone who passes the gate sees all of it, and the gate is the whole control. Check that it covers every path (pages, data files, assets, source maps), that the host's default and preview addresses cannot bypass it, that no secrets are in the bundle, and that the credentials used to pull data at build time stay in CI. Once it needs writes, per-user or per-role data, or live data, it moves to P1 below.
  - **P1 Validating.** A browser-only app (an SPA or static site that reaches the database or third-party APIs directly) is the main risk: every rule lives in the client or in database policies. Server-side rendering, or an SPA plus a lightweight backend that owns the database connection, verifies the caller and holds the secrets, is usually enough.
  - **P2 Building the real thing** (the MVP after validation). It will need hardening, so a dedicated backend and an established framework to carry it. Lean towards fully featured frameworks on the backend and frontend (auth, validation, migrations, jobs and testing built in or first-party): they keep more options open as the product grows.
  - **P3 Scaling workloads.** Data pipelines, background jobs and queues, or workarounds standing in for them (cron hitting HTTP routes, long-running requests, polling tables as a queue), mean it is time for a major cloud provider (AWS, GCP or Azure) rather than only an app platform such as Railway or Render.
  - **At every stage:** the database is reached only through a backend, lightweight at P1 and dedicated from P2. The browser never connects to it directly. The backend verifies the caller and connects with a least-privilege credential; secrets stay on the server; the most privileged key is used only in scheduled and admin jobs. Database access rules (such as row-level security) are welcome as a second layer, not the only one. Where an app already reaches the database from the browser with full access rules (Supabase Route A), it is secure and not a finding: recommend the move to a backend as the next step for its stage.
- **Identity.** One sign-in provider, roles or groups for permissions, account removal. For internal tools: a single-tenant setup and leaver handling.
- **Delivery.** Quality gates on every push, migrations in code applied automatically before deploy, a build-once artifact promoted through environments, a pre-production environment (staging and test mean the same thing: if one exists, keep it; if not, add one soon) with its own database and secrets, no more than two environments in total for most teams at this stage because a third is too many to keep in step, a manual or approved production trigger, feature flags where releases are risky, configuration through environment variables.
- **Observability.** Sentry or an equivalent on every tier, central structured logs, alerts routed to a named owner by environment and criticality, job heartbeats.
- **Ownership.** Code and accounts under the company, not personal accounts or one laptop.
### Architecture level (0 to 5), for the scorecard
Score the system as the **weakest** of four pillars: connection model, where secrets live, access control, and identity/ownership. One weak pillar caps the score.

| Level | Meaning |
|---|---|
| 0 Exposed | Secrets in the browser or repo, or sensitive data reachable with no auth. |
| 1 Client-trust | SPA or static site reaches the database or third-party APIs directly; security lives in the client; weak or missing data rules. |
| 2 Browser-direct, enforced | Browser talks to the database directly, but every table or route has real server-side rules (full RLS or a gateway); secrets server-side. Supabase Route A done right. |
| 3 Server boundary | A backend (even lightweight) owns data access and verifies the caller; least-privilege where it touches data; one sign-in provider with roles. |
| 4 Hardened | Dedicated backend and an established framework; least-privilege connections; environments and ownership in order; roles or ownership and leaver handling. |
| 5 Scales cleanly | Workloads split out (jobs, queues, pipelines) on a real cloud where needed; portability (ORM or abstraction); defence in depth (boundary and data rules); nothing privileged in the request path. |

Target by product stage: **P1 -> 2, P2 -> 3, P3 -> 4** (each is also the minimum; below it is a finding). **Sensitivity bump:** if the app holds special-category, health or financial data at scale, raise the target by one. Report architecture as `level / target` with a light, same as the other dimensions.

Say what to do now, what to do when the team, the product stage or the sensitivity of the data grows, and what to leave alone. Label stage items as pragmatic suggestions, not security advice.

## Recommendations
Ordered by work, not by finding number. Each has an effort tag (quick fix, days; weeks; housekeeping; later project). Open with a short paragraph naming real good practice, then a "Suggested order" line. Do not interleave paragraphs inside the numbered list.

## What's working well
A short list of controls that hold, each checked (live, code review, dependency audit).

## To verify
Each unverified item as: **To verify:** do this (where). **Expect:** result. **If not:** meaning and next step.

## Coverage and limits
What was and was not examined, tools used, anything that could not be reached. State that live settings were not checked unless the user supplied results.

## Categories (one primary category per finding: where the fix lives)
Severity says how bad; category says what kind of problem. Use exactly these eight names.
1. **Access control:** sign-in, who counts as an authorised user, roles and permissions, sessions, account removal, admin pages.
2. **Data protection:** personal or sensitive data exposed, over-sent, stored or retained too widely, in logs, repos or exports.
3. **Secrets and keys:** hard-coded or shared secrets, tokens and keys held in the browser or repo.
4. **App security:** injection, untrusted content (including AI input) steering actions, headers and browser protections, abuse and rate limits.
5. **Dependencies:** vulnerable, unpinned or end-of-life packages.
6. **Quality gates and deployment:** CI gates, migrations, environments, how releases reach production.
7. **Resilience and monitoring:** backups, availability, failure handling, third-party service reliability, logging, error reporting, alerting, audit trail.
8. **Housekeeping:** ownership of code and accounts, third-party access, documentation, accepted risks, tidy-ups.

Optional tags, kept light: **Fix type** (config, code, process, decision), **Evidence** (Live, Code, Docs, Unverified), and the **pragmatic deferral** flag. Report a count by category in the overview.

