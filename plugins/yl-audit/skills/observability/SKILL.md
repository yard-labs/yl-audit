---
name: observability
description: Read-only audit of how an application is observed in production: centralised logging, error reporting (Sentry preferred, equivalents such as Bugsnag, Rollbar, Honeybadger, Datadog, New Relic, Highlight, PostHog accepted), alerting, uptime and health checks, job and cron monitoring, whether calls to third-party services fail safely, and whether logs leak sensitive data or errors are silently swallowed. Levels observability 0 to 5 against what suits the team's stage, and gives ordered steps plus "to verify" checks for the settings the repo cannot show. Use when reviewing whether failures would be noticed, diagnosed and acted on.
argument-hint: [path to repo] [optional: team size and stage, e.g. "solo, early" or "5 people, with a test environment"]
---

# Observability audit

Ask one question: if this app broke at 2am, or quietly started losing data, would anyone find out, and could they work out why? Code and config show what is wired up. Settings in Sentry, the log platform and the alert tools show whether it works, and those are for the user to verify.

## Rules

Follow `../full/references/rules.md`. On top of it:
- Never send a test error or trigger an alert.
- Sentry DSNs, auth tokens and API keys: name the variable and where it is referenced. A browser DSN is public by design but still not repeated in the report.
- Settings in external tools are unverified until the user checks (To verify lines from `references/settings-checks.md`). Never assume an alert exists because the SDK is installed.

## What good looks like

Expected by default; the stage table in `references/practices.md` says when an item is optional.

1. **Centralised logging.** Structured logs (JSON, levels, request or correlation IDs) shipped off the host to a central store (Axiom, Datadog, Grafana Loki, Better Stack, CloudWatch, Vercel log drains or equivalent), with retention long enough to investigate an incident and access limited to the people who need it. No sensitive data in logs: personal data, tokens, email or document bodies, full request and response bodies, raw model output.
2. **Error reporting.** **Sentry preferred**, or an equivalent. On every tier: front end, back end and routes, scheduled jobs and workers. Environment and release tagged, source maps uploaded in CI but not served publicly, personal data off by default and scrubbed, error boundaries in the UI, unhandled rejections captured, failures of external and AI calls captured with context.
3. **Alerting.** Rules that reach a named person or channel: new and regressing errors, error-rate spikes, failed or missed scheduled jobs (a heartbeat or cron monitor), uptime and health checks, resource limits, and cost spikes (AI, paid APIs). Production and critical alerts land somewhere the owner reads, apart from staging and informational noise.
4. **Health and uptime.** A health endpoint and an external uptime check that alerts.
5. **Audit trail kept separate** from operational logs: who did what, tamper-resistant, retained.
6. **Deploys visible in errors.** Releases recorded so a new error can be tied to a deploy.
7. **Tracing and metrics.** Optional early; expected for teams (OpenTelemetry or the vendor SDK).
8. **No silent failures.** Errors are reported, not swallowed (`catch {}`, log-and-continue, "mark done on failure").
9. **Third-party calls fail safely.** Calls to outside services on core flows have a timeout, a bounded retry or fallback, handle rate limits (`429`, `Retry-After`), and pin the API version where the vendor makes breaking changes.

## Levels

| Level | Observability |
|---|---|
| 0 | Nothing, or console output only |
| 1 | Platform logs only (short retention, no search, no alerts); no error reporting |
| 2 | Error reporting on at least one tier |
| 3 | Error reporting on every tier with environment and release tags and source maps, plus structured logs in a central store |
| 4 | Level 3 plus alerting to a named owner (error spikes, job heartbeats, uptime) and scrubbing of sensitive data |
| 5 | Level 4 plus traces or metrics, deploy markers, cost and security anomaly alerts, tested alerts, and an incident note or runbook |

Report the level reached and the level **right for the stage**. A solo early-stage app at level 4 (Sentry on every tier plus alerts to one person) is in a good place. A team at level 2 is not.

## Procedure

1. **Intake.** Ask once: team size and stage, hosting, tools they already use or pay for, who is on call, known findings, and the two alert delivery questions in `references/settings-checks.md`.
2. **Inventory.** Dependencies and imports (`@sentry/*`, `sentry-sdk`, `bugsnag`, `rollbar`, `honeybadger`, `@datadog/*`, `dd-trace`, `newrelic`, `@opentelemetry/*`, `pino`, `winston`, `bunyan`, `structlog`, `loguru`, `@axiomhq/*`, `logtail`, `@highlight-run/*`, `posthog`), config files (`sentry.*.config`, `instrumentation.ts`, `sentry.properties`, OTel config, log drains in hosting config), environment variable names (`SENTRY_DSN`, `SENTRY_AUTH_TOKEN`, `DD_API_KEY`, `LOG_LEVEL`), CI steps (`sentry-cli releases`, source map upload), health routes, and monitoring workflows.
3. **Map the flow.** Draw a plain-text diagram: each tier of the app, where its logs and errors go, the destinations, the alert routes and the person or channel at the end. Mark each element Code, Config or Docs. Tiers with nowhere to go are called out.
4. **Evaluate** each check in `references/practices.md` with `file:line` evidence: pass, partial, fail, n/a for stage, or unverified.
5. **Silent failures, sensitive logging and third-party calls.** Run the scans in `references/practices.md` (swallowed errors, jobs that report success after a failure, log calls that include personal data, tokens, bodies or model output, error text returned to callers). Count and cite; never print the logged values.
6. **Level and target.** Report both.
7. **Settings to verify.** Build the table from `references/settings-checks.md` for everything the repo cannot show. Record the alert delivery answers as Docs evidence, unverified until the user confirms the integration settings.
8. **Report** in the shared findings format (`../full/references/report-template.md`): overview, observability diagram, check table, a third-party services table (Service | Used for | Timeout | Retry or fallback | API version pinned | Rate limit handling | To verify), findings (ID, name, finding, criticality, risk, remediation) High to Low, ordered recommendations with effort tags and a suggested order, what is working well, the To verify list, and coverage limits.
