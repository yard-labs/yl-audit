# Settings to verify (the repo cannot show these)

Give the user these checks; do not run them. Confirm names and paths against current vendor documentation. Each unverified item uses: **To verify:** do this (where). **Expect:** result. **If not:** meaning and next step.

## Error reporting (Sentry shown; equivalent settings elsewhere)
| Setting | Where | To verify: do this | Expect |
|---|---|---|---|
| Projects and environments | Sentry > Projects, Settings > Environments | List projects and environments | A project per tier (or per app), environments for production and staging, no mixing |
| Events are arriving | Sentry > Issues, filter by environment and time | Check recent events for each tier | Recent events from front end, back end and jobs (or a known reason why not) |
| Alert rules | Sentry > Alerts | Open the rules | New issue, regression and spike alerts for production; a cron or job monitor alert if scheduled jobs exist |
| Alert routing | the rules' actions and Settings > Teams | Read where each alert goes | A named person, team or channel that is actually read; not an unmonitored inbox |
| Issue ownership | Settings > Ownership | Read the rules | Owners assigned, or one owner for a solo project |
| Data scrubbing | Settings > Security and Privacy | Read server-side scrubbing, default PII, IP storage | Scrubbing on; sensitive field names listed; personal data off by default |
| Release and source maps | Sentry > Releases | Open the latest release | The release matches a deploy; source maps attached |
| Quotas and spike protection | Settings > Subscription / Spike protection | Read quota and limits | Enough headroom; spike protection on; a budget alert |

## Logs
| Setting | Where | To verify: do this | Expect |
|---|---|---|---|
| Central destination | log platform or the host's log drains | Confirm logs from every tier are arriving | Front end, back end, jobs, and database logs are searchable in one place |
| Retention | log platform settings | Read retention | Long enough to investigate an incident (days to weeks at least) |
| Access | log platform members | List who can read logs | Only people who need it (logs may hold sensitive data) |
| PII filters | log platform or agent | Read any redaction rules | Sensitive fields masked before storage |

## Alerts and uptime
| Setting | Where | To verify: do this | Expect |
|---|---|---|---|
| Uptime monitor | uptime tool (Better Stack, UptimeRobot, Checkly, host) | List monitors | A check on the health route and the main page, from outside, with alerting |
| Job heartbeat | cron monitor or heartbeat tool | List monitors for each scheduled job | One per job, with a missed-run alert |
| Database and host metrics | database and host dashboards | Read alerts | CPU, memory, disk and connection alerts to a person |
| Cost alerts | AI and paid API consoles | Read spend limits and alerts | A budget alert and a hard limit |
| Test the path | any alert | Ask the owner to trigger a safe test alert | It arrives, to the right person, within minutes |
| On-call | team agreement | Ask who is told and what they check first | A named person and a one-page incident note (S2 and above) |

## Alert delivery (ask the user at intake; record the answers as Docs evidence)
1. Where do production alerts go today (Slack, Teams, Discord, email, paging tool), and who reads them?
2. Are production and critical alerts kept apart from staging and informational noise (deploys, digests)?

Expected, by stage: **S1** one production destination the owner reads, critical alerts also by email or push. **S2** separate production and non-production channels, a critical versus normal split, an owner per channel. **S3** channels by environment and criticality, paging with escalation, a named rota.

| Setting | Where | To verify: do this | Expect |
|---|---|---|---|
| Integration connected | each source's notification settings (error tracker, uptime, cron monitors) | Open each source's notification settings | Each source posts to the intended destination |
| Webhook storage | chat tool integrations and the secrets store | Confirm webhook addresses are held as secrets | Not in the repo, not in logs, rotated if ever exposed |
| End-to-end test | any alert source | Ask the owner to trigger a safe test alert | Arrives in the right place, with context, within minutes |
