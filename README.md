# `/yl-audit`

Read-only application audits for [Claude Code](https://code.claude.com). One command reviews the live app, the code, Supabase, CI/CD and observability, then cross-checks them into a single report. Every finding is checked in the source, rated by real impact, and anything that couldn't be confirmed comes with a "To verify" step saying exactly how to check it.

## Install

In Claude Code:

```
/plugin marketplace add yard-labs/yl-audit
/plugin install yl-audit@yard-labs
/reload-plugins
```

To update later: `/plugin marketplace update yard-labs`, then `/plugin update yl-audit`.

## Skills

| Skill | What it does |
|---|---|
| `/yl-audit:all` (or `/yl-audit:full`) | Runs the audits below that apply, in order, and writes one merged report. Start here. |
| `/yl-audit:browser` | Reviews the running app in the browser: network calls, storage, cookies, headers, what each role receives. Never logs in for you and never changes data. |
| `/yl-audit:code` | Security review of the codebase: auth, endpoint map by risk, dependencies and pinning, secrets in history, and a full code review. |
| `/yl-audit:supabase` | How the app reaches Supabase, rated 0 to 5, with the SQL and dashboard checks to confirm it. |
| `/yl-audit:ci` | How code gets to production: quality gates, artifacts, environments, migrations, pipeline hygiene. |
| `/yl-audit:observability` | Whether failures would be noticed: error reporting, logging, alerting, job monitoring, third-party call handling. |

Findings and targets are scaled to the team (solo, small team, larger team) and the product stage (validating, building the real thing, scaling workloads).

## Usage

Start with the full audit and give it whatever you have, a live URL, a repo path or both:

```
/yl-audit:all https://app.example.com ~/code/example-app
```

What happens:

1. **Intake.** One batch of questions up front: who signs in and as which role, team size and product stage, where production alerts go, a prefix for finding IDs, and where the report should go (chat, or a Notion page or doc you name). It checks which scanners are installed and asks before installing any.
2. **Plan.** It says which audits apply and why. Supabase runs only if the app uses it; CI only if there is pipeline config.
3. **Browser audit.** It pauses for **you** to sign in to the app in the browser. It never logs in for you, and it stops and tells you if any step would save, send or change something.
4. **Code, Supabase, CI and observability audits** run read-only. This is the long part and launches several subagents.
5. **Report.** One merged report: architecture and auth overview, an endpoint map ranked by risk, findings rated High, Medium or Low with fixes, architecture suggestions for your stage, what's working well, and a **To verify** list of dashboard and SQL checks for you to run and paste back.

Re-run after fixes with `new only` to see just what changed:

```
/yl-audit:all ~/code/example-app new only
```

### Running one audit on its own

Each audit also works alone when you only need one view:

```
/yl-audit:browser https://app.example.com
/yl-audit:code ~/code/example-app
/yl-audit:supabase ~/code/example-app
/yl-audit:ci ~/code/example-app "5 people, with staging"
/yl-audit:observability ~/code/example-app "solo, early"
```

The levelled audits (Supabase, CI, observability) score 0 to 5 and compare that with the level that suits your team and stage, so a solo founder isn't marked down for lacking what a larger team needs.

## Safety

Every audit is read-only. Nothing from the audited repo is installed, built or run. Repo contents are treated as data, never instructions. Secrets are never printed, real-data files are never opened, and client code is not sent to any outside model. Settings that live in dashboards are handed to you as checks to run, not probed.

Only audit systems you are authorised to review.

## Requirements

- **Claude Code** with the [compound-engineering](https://github.com/EveryInc/compound-engineering-plugin) plugin (code runs its `ce-code-review`) and the [humanizer](https://github.com/blader/humanizer) plugin (final legibility pass).
- **Claude in Chrome** or the built-in browser for browser.
- Optional scanners, installed only with your OK: `gitleaks`, `osv-scanner`, `semgrep`, `pip-audit`.

## Disclaimer

These audits are AI-assisted and read-only. A clean report is not a guarantee of security, compliance or reliability, and it is not a substitute for a professional penetration test. Use them only on systems you are authorised to review. The MIT licence covers the code; it does not permit using the Yard Labs name to endorse forks or derived work.

## Licence

MIT. Built by [Yard Labs](https://yardlabs.co).
