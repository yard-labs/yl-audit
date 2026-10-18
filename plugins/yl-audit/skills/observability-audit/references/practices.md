# Practice checks

Rate each: **pass**, **partial**, **fail**, **n/a for stage**, or **unverified**. Cite `file:line`.

Stages S1, S2 and S3 are defined in `../../app-audit/references/rules.md`.

## Checks

| # | Check | What to look for | S1 | S2 | S3 | Typical rating if absent |
|---|---|---|---|---|---|---|
| O1 | Error reporting on the back end and routes | Sentry or equivalent initialised server-side (`Sentry.init`, `sentry_sdk.init`, instrumentation file) | Expected | Expected | Expected | Medium |
| O2 | Error reporting on the front end | SDK in the client bundle; error boundaries; unhandled rejection capture | Expected | Expected | Expected | Medium |
| O3 | Error reporting in jobs, cron and workers | SDK initialised in scripts and scheduled entry points; cron monitor or heartbeat | Expected | Expected | Expected | Medium |
| O4 | Environment and release tags | `environment`, `release` set; release created in CI | Expected | Expected | Expected | Low |
| O5 | Source maps | uploaded to the error tool in CI; not served publicly | Expected | Expected | Expected | Low (public maps: Low to Medium) |
| O6 | Personal data off by default and scrubbed | `sendDefaultPii` false; `beforeSend` or scrubbing rules; no user email in context unless needed | Expected | Expected | Expected | Medium |
| O7 | Structured central logging | a logger library, JSON output, levels, request or correlation IDs, shipped to a central store (drain, agent or SDK) | Optional (platform logs acceptable) | Expected | Expected | Low (S1), Medium (S2+) |
| O8 | Log retention and access | retention fits incident work; access limited | Expected | Expected | Expected | Low |
| O9 | No sensitive data in logs | no email or document bodies, tokens, full request or response bodies, raw model output, personal data in log calls | Expected | Expected | Expected | Medium (High if tokens or credentials) |
| O10 | No silent failures | no swallowed errors on I/O; failed jobs are not marked done; errors reach the tracker | Expected | Expected | Expected | Medium |
| O11 | Alert rules | rules exist for new errors, error spikes, job failures, uptime | Expected (to one person) | Expected | Expected | Medium |
| O12 | Alert routing and owner | goes to a named person or channel; escalation for S3 | Expected | Expected | Expected | Medium |
| O13 | Uptime and health | a health route and an external monitor that alerts | Expected | Expected | Expected | Medium |
| O14 | Job and cron monitoring | heartbeat or monitor per scheduled job; alert on a missed run | Expected | Expected | Expected | Medium |
| O15 | Resource and cost alerts | database CPU, memory, disk; AI and paid API spend; quota limits | Optional | Expected | Expected | Low |
| O16 | Auth and security anomaly alerts | spikes in 401 or 403, failed sign-ins, new admin actions | Optional | Optional | Expected | Low |
| O17 | Audit trail | who-did-what events stored separately, tamper-resistant, retained | Expected where there is sensitive data | Expected | Expected | Low to Medium |
| O18 | Deploy markers | releases visible in the error tool; deploy events recorded | Optional | Expected | Expected | Low |
| O19 | Tracing and metrics | OpenTelemetry or vendor performance monitoring | Optional | Optional | Expected | none (a win when present) |
| O20 | Incident readiness | a note on who is told and what to check first | Optional | Expected | Expected | Low |
| O21 | Alert delivery | alerts reach a chat, paging or email destination the owner reads; production and critical alerts kept apart from staging and informational noise | Expected (one place the owner reads) | Expected | Expected, with escalation | Medium (alerts go nowhere), Low (not separated) |
| O22 | Webhook addresses | chat webhook URLs (Slack, Teams or Discord incoming webhooks) held as secrets, not in the repo or logs | Expected | Expected | Expected | Medium (committed; anyone with it can post as the integration) |
| O23 | Third-party calls | timeout, bounded retry or fallback, `429` handling, API version pinned, on calls to outside services in core flows | Expected | Expected | Expected | Medium (core flow, no timeout, production), Low otherwise |

## Scans (counts and locations only; never print logged values)
- **Swallowed errors:** empty `catch {}`, `catch (e) { console.* }` with no rethrow or report, `.catch(() => {})`, `except: pass`, functions that `return null` on error, handlers that mark work done in a `finally` or after a failure.
- **Sensitive log calls:** `console.log|error|warn`, `logger.*`, `print(` whose arguments include `email`, `token`, `password`, `body`, `content`, `prompt`, `response`, `req`, `headers`, `authorization`, `JSON.stringify(` of a whole object, or full error stacks returned to callers.
- **Error text returned to callers:** `error.message` or exception strings placed in HTTP responses.
- **Logging convention:** a rule against `console.*` in docs or lint config, and whether code follows it.
- **Health routes:** `/health`, `/healthz`, `/api/health`, readiness checks that touch dependencies.
- **Source maps:** build config that emits or uploads maps (`productionBrowserSourceMaps`, `sourcemap: true`, `sentry-cli sourcemaps`).
- **Chat notification steps and webhooks:** `slackapi/slack-github-action`, `slack`, `hooks.slack.com`, `discord.com/api/webhooks`, `outlook.office.com/webhook` or `webhook.office.com`, `incoming-webhook`, PagerDuty or Opsgenie keys, and whether the address is a secret reference (`${{ secrets.X }}`, an environment variable) or a literal. Count and cite; never print the address.
- **Third-party calls:** list every outside service from code and config (SDK imports, base URLs, hosts in `fetch` and `axios` calls, `*_API_KEY`, `*_URL`, `*_TOKEN` variable names, CDN script tags, content-policy hosts), grouped (identity, data, AI, email, payments, analytics and monitoring, other). For each: timeout (`AbortSignal.timeout`, a client `timeout` option, SDK setting), bounded retry or fallback, `429` and `Retry-After` handling, API version header or path (`anthropic-version`, `api-version`, `/v1/`). Cite the line. Vendor limits and plans are not in the code: To verify in the vendor dashboard.
- **Scheduled jobs:** cron definitions (workflows, `vercel.json`, database cron, queue workers) with no matching monitor.

## Severity cues
- **High:** tokens, credentials or secrets written to logs or sent to the error tool.
- **Medium:** production app with no error reporting; failures swallowed so data is lost silently; scheduled jobs with no missed-run alert; personal data in logs or error reports; alerts that go nowhere.
- **Low:** a third-party call with no timeout where the failure only delays a background job; API version not pinned; no `429` handling where quotas are loose; no central logging for a solo early-stage app; no deploy markers; no tracing; short retention; no incident note.
- Items rated down because of stage get the pragmatic-suggestion label from the rules.
