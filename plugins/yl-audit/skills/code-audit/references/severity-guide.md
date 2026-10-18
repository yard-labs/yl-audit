# Severity guide

Rate by real consequence to this client, not by the scanner's label.

## High
- A live secret, key or credential reachable by an outsider or committed (including third-party API keys that can bill).
- Large sets of personal data committed or exposed.
- A route that sends a privileged token to an attacker-chosen address, or lets an outsider read or write data.
- No backups for data that exists nowhere else.

## Medium
- Weakness reachable by any signed-in user or any shared-secret holder that changes or exposes data (missing role or ownership checks on sensitive data, shared static secrets, identity taken from the request instead of the session).
- Untrusted text (user input, emails, documents, web pages) steering AI actions that change records.
- A system that loses or mangles input silently when something fails.
- Known-vulnerable dependencies on shipped code where the affected feature is not used (still Medium: upgrade is due).
- Tokens or admin secrets held in the browser with no security headers.
- Missing delivery controls that let unchecked changes reach production.

## Low
- Hardening gaps, information leaks in errors and logs, test and doc hygiene, staleness.
- **Stage trade-offs** (no roles, no rate limits) when the users are a small, known group, for example an internal tool. Rate Low with the pragmatic-suggestion label from `../../app-audit/references/rules.md`. Keep cheap fixes (for example timeouts) in the remediation.

## Rules of thumb
- A critical-labelled package whose vulnerable feature is not used is Medium, not High. Say what you checked.
- Latent issues (not reachable today) are Low unless one other fix would make them reachable.
- A client's own documented, accepted risk is still a finding; say it is known.
- Always record real good practice. A report that is only problems reads as inaccurate.
- If two findings share one cause (flat policy and no roles), merge them or cross-reference; do not double count.
- Never rate on a claim you have not opened in the code.

## Dependency pinning
- No lockfile for a shipped app, or a deploy that ignores it (`npm install` instead of `npm ci`): Medium. Builds are not reproducible and a poisoned release can be pulled in silently.
- Python with unpinned or missing requirements (so it cannot be audited): Medium if it runs in production or against production data, Low for local one-off scripts. Say which.
- Git or URL dependencies, `latest`, wildcard versions, install scripts from unfamiliar packages: Low to Medium by what runs.
- CDN scripts without a version or integrity hash: Low. (Actions, images and Dependabot are rated by `yl-audit:ci-audit`.)
- A tool that was skipped means the result is **unverified**, never clean.

## Library health
- A framework or runtime past end of support in production: Medium (security fixes stop). Confirm the date against the vendor's policy.
- Deprecated or abandoned packages: Low, higher if shipped and security-relevant.

