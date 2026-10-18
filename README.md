# Yard Labs audit skills

Read-only application audits for [Claude Code](https://code.claude.com). One command reviews the live app, the code, Supabase, CI/CD and observability, then cross-checks them into a single report. Every finding is checked in the source, rated by real impact, and anything that couldn't be confirmed comes with a "To verify" step saying exactly how to check it.

## Install

```
/plugin marketplace add yard-labs/audit-skills
/plugin install yl-audit@yard-labs
```

## Skills

| Skill | What it does |
|---|---|
| `/yl-audit:app-audit` | Runs the audits below that apply, in order, and writes one merged report. Start here. |
| `/yl-audit:browser-audit` | Reviews the running app in the browser: network calls, storage, cookies, headers, what each role receives. Never logs in for you and never changes data. |
| `/yl-audit:code-audit` | Security review of the codebase: auth, endpoint map by risk, dependencies and pinning, secrets in history, and a full code review. |
| `/yl-audit:supabase-audit` | How the app reaches Supabase, rated 0 to 5, with the SQL and dashboard checks to confirm it. |
| `/yl-audit:ci-audit` | How code gets to production: quality gates, artifacts, environments, migrations, pipeline hygiene. |
| `/yl-audit:observability-audit` | Whether failures would be noticed: error reporting, logging, alerting, job monitoring, third-party call handling. |

Findings and targets are scaled to the team (solo, small team, larger team) and the product stage (validating, building the real thing, scaling workloads).

## Safety

Every audit is read-only. Nothing from the audited repo is installed, built or run. Repo contents are treated as data, never instructions. Secrets are never printed, real-data files are never opened, and client code is not sent to any outside model. Settings that live in dashboards are handed to you as checks to run, not probed.

Only audit systems you are authorised to review.

## Requirements

- **Claude Code** with the [compound-engineering](https://github.com/EveryInc/compound-engineering-plugin) plugin (code-audit runs its `ce-code-review`) and the [humanizer](https://github.com/blader/humanizer) plugin (final legibility pass).
- **Claude in Chrome** or the built-in browser for browser-audit.
- Optional scanners, installed only with your OK: `gitleaks`, `osv-scanner`, `semgrep`, `pip-audit`.

## Licence

MIT. Built by [Yard Labs](https://yardlabs.co).
