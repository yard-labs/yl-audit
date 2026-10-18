---
name: browser-audit
description: Read-only review of a running web app through the browser. Observes what the app actually sends, stores and exposes: network calls and API endpoints, direct database calls, auth tokens and cache in browser storage, cookies, security headers, CORS, what changes after sign-out, and what data each role receives. Signed-out and signed-in passes, never logging in for you, never changing data. Use when you have a live URL to review, alongside code-audit, or when no code was supplied.
argument-hint: [live URL] [optional: role you are signed in as]
---

# Browser audit

Review the app as a user's browser sees it. This finds what the code review cannot: what really goes over the wire, what really sits in storage, what the headers really say, and what each role really receives. It feeds the endpoint map used by the `yl-audit:code-audit` skill.

Before the first browser step, read the `chrome-browser` skill (Claude in Chrome) or the `built-in-browser` skill, whichever tool is in use, and follow its tool-loading and safety rules.

## Rules

Follow `../app-audit/references/rules.md`. On top of it:

- **No write actions in the browser.** Do not do any write, edit or create action. If anything comes close (a button that saves, submits, deletes, sends, creates, approves, or a form), **stop, and tell the user where you stopped: the path and the action.** Do not trigger alerts, confirms or dialogs.
- **Never log in for the user.** Do not enter credentials, create accounts, solve CAPTCHAs, accept terms or grant permissions. Do not try to bypass sign-in or an access gateway. The user signs in; you work in their session only when they ask you to.
- **Do not copy customer data.** Observe structure, not values: field names, counts, status codes, sizes. Never transcribe records or personal data from a screen or a response. Note that a screen or response contains personal data, and what kind.
- **Tokens and keys:** report type, presence, location and claim names only, never the value.
- **Never call an API route by hand unless the code shows it is a pure read.** Some GET routes run jobs, send mail or spend money. Prefer watching what the app itself calls. Never send POST, PUT, PATCH or DELETE requests. Observing the app's own requests to third parties is fine.
- **Keep an audit trail.** Record each page visited and each action taken (path and action), so the user can see exactly what was done.
- **Stay on task.** If a page will not load, an element does not respond or the work grows beyond the plan, stop and ask. Do not loop.
- **If the browser tool is not available or not connected, stop and ask** the user to enable or connect it (or whether to install the extension). Do not substitute another route or skip silently. If they decline, report the browser audit as not run and everything it would have shown as unverified.

## Procedure

1. **Prep.** Confirm the URL(s), authorisation, and which role the user is signed in as. Read any endpoint map or architecture overview from `yl-audit:code-audit`. Agree the stopping rule above.
2. **Signed-out pass** (no credentials). Use `references/browser-checklist.md`.
   - Load the root and a few deep links. Record redirects (sign-in provider, access gateway).
   - Response headers on the page and a static file: content security policy, frame protection, HSTS, content-type options, referrer policy, permissions policy, CORS, cache headers.
   - Cookies and their flags. Source maps, `.env`, `.git`, package files and unknown paths (GET only, static paths).
   - What the page shell and bundles contain: counts of keys and project addresses by pattern (never values), public environment variables.
   - Admin or internal pages reachable without sign-in (open the page, do nothing on it).
3. **Signed-in pass** (the user signs in, or you use their existing session at their request). Browse the main screens read-only.
   - **Network log:** every request: method, host and path, status, whether it carries credentials (yes or no), and the response shape by field names. Mark which go to the app's own API, directly to the database or storage, and to third parties.
   - **Storage:** names, sizes and kinds of local storage, session storage, IndexedDB and cookies. Tokens: presence, kind (identity, access, refresh), scopes and claim names, never values.
   - **Data exposure:** compare what the screen shows with what the response contains (columns returned but hidden, `select=*`, financial or personal fields sent to every viewer).
   - **Console:** errors and warnings that reveal internals.
   - **Sign-out test** (only if the user agrees): sign out, then check storage, the back button and cached screens for left-over data.
4. **Role comparison** (if the user can switch roles or has a second account): compare screens and responses. Do not create or change roles.
5. **Compare with the code.** Mark each observed endpoint in the endpoint map. Flag endpoints the code review missed, and gates the docs claim but the traffic does not show.
6. **Report** (below). Evidence is tagged **Live** and described as "Seen in the browser's network log" or similar. Anything you could not see gets a **To verify: do this** line.

## Report

1. **Observed architecture:** the browser's view of what talks to what, with the credential on each connection.
2. **Endpoints observed:** a table bucketed by auth mechanism and ranked by data risk exposure (same scale as `yl-audit:code-audit`'s endpoint map).
3. **Headers table:** each header, value present or absent, and what it means.
4. **Storage table:** what is held in the browser, for how long, and what survives sign-out.
5. **Findings** in the `yl-audit:code-audit` format (ID, name, finding, criticality, risk, remediation), High to Low, with stage trade-offs labelled as pragmatic suggestions.
6. **What is working well.**
7. **Audit trail:** pages visited and actions taken.
8. **Coverage and limits:** what was not reached, and any point where you stopped.
