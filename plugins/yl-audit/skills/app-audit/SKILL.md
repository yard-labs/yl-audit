---
name: app-audit
description: Run a full read-only security audit of an application end to end by orchestrating browser-audit (live app), code-audit (source), supabase-audit (only if Supabase is used), ci-audit (only if CI/CD config exists) and finally observability-audit (logging, error reporting and alerting), then cross-checking the results and writing one report. Use when you have a live URL, a codebase, or both, and want a single calibrated findings report with every unverified item paired with a way to verify it.
argument-hint: [live URL] [path to repo] [optional: known findings file, or "new only"]
---

# App audit

One run, one report. Order: **browser, then code, then Supabase if it is relevant, then CI/CD if there is pipeline config, and observability last**.

## Rules

- Follow `references/rules.md`. Every audit below follows it too; where an audit adds its own rules, the strictest applies.
- **Browser actions are read-only.** Never write, edit or create anything in the live app. If a step comes close, stop and tell the user the path and the action. Never log in for the user. The user signs in.

## Procedure

1. **Intake (one batch of questions, up front).**
   - Live URL(s), and who will sign in and as which role.
   - Repo path(s). Whether history is included.
   - Known findings to exclude, or "new only".
   - An ID prefix for findings (for example `ACME`).
   - Output: in the chat, or a named Notion page or document.
   - Whether the user can run the Supabase and settings checks and paste results back.
   - Team size and stage (S1, S2 or S3 in the rules), product stage (P1, P2 or P3), and whether pull requests are used.
   - Where production alerts go today, and who reads them.
   - Tools: check which scanners (gitleaks, osv-scanner, pip-audit, semgrep) and which browser tool are available, and ask about anything missing in this batch (see Tools in the rules).
   Warn that the run is long and will launch several read-only subagents. After this, only stop for the browser sign-in, a stop condition, or a real blocker.
2. **Plan.** State which audits apply and why: browser if there is a URL; code if there is a repo; Supabase if either shows it (a Supabase address or `/rest/v1`, `/auth/v1` or `/storage/v1` in traffic, `@supabase/*` packages, a `supabase/` folder, Supabase keys or addresses in config). Say what is out of scope. Keep one line of progress between phases.
3. **Browser audit** (if a URL). Invoke `yl-audit:browser-audit`. The user signs in; you do not. Keep its audit trail. Collect: observed architecture, endpoints seen (with credential kind), headers, storage, data over-sent, findings tagged **Live**, and any Supabase traffic.
4. **Code audit** (if a repo). Invoke `yl-audit:code-audit`, passing the browser's observations (endpoints seen, headers, storage, Supabase traffic) so the endpoint map starts with real traffic, plus the known findings. Collect: architecture, authentication mechanism, endpoint map, dependency and pinning results, secrets and history, review findings, and the verified findings.
5. **Supabase audit** (only if relevant). Invoke `yl-audit:supabase-audit` with the code, the endpoint map and the browser's Supabase observations (which key, which calls, what the bundle held). Collect: level 0 to 5, delivery checks, anon exposure, and the settings table with its To verify lines.
6. **CI audit** (only if the repo has pipeline config such as workflows, Dockerfiles or hosting config). Invoke `yl-audit:ci-audit` with the stage from intake, the code audit's findings, the Supabase audit's delivery checks (migrations in code, applied in CI, RLS checks), and any pipeline facts the browser saw (deployed version, headers). Collect: the pipeline diagram, level and target level, check table, findings, and the settings to verify.
7. **Observability audit (always last).** Invoke `yl-audit:observability-audit` with the stage and alert answers from intake, the code audit's findings on logging, swallowed errors and third-party calls, the CI audit's view of releases and source maps, the Supabase audit's view of log retention, database metrics and alert settings, and anything the browser saw (front-end SDK loading, requests to an error tool, public source maps). Collect: the observability diagram, level and target level, check table, findings, and the settings to verify.
8. **Cross-check and merge.** Follow `references/merge-and-crosscheck.md`: reconcile the audits, mark corroboration, flag contradictions, merge the endpoint map, consolidate the To verify items, deduplicate and rate.
9. **Report.** One report in the code-audit template, extended per the merge reference, then the legibility pass from the rules (once, on the merged report). Deliver it where the user said.
10. **Close.** List what is still unverified and who has to act (client dashboard, project owner, the user running SQL or commands). Offer a "new only" re-run after fixes.

## If something is missing

- **URL only:** run the browser audit; report architecture and endpoints as observed; mark everything the code would show as unverified; say a code drop would verify it.
- **Code only:** run code audit, and Supabase if relevant; mark everything the running app would show (headers, storage, real responses) as unverified and give the browser checks.
- **Neither can run** (no sign-in, no access): stop and say what is needed.
