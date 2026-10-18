# Endpoint map

Build one table for the whole system. An "endpoint" is anything a caller can invoke: HTTP routes (Next.js route handlers and pages/api, Express routers, serverless and edge functions, workers), scheduled and webhook entry points, and, where the browser talks to the database directly, every table, view, function (RPC) and storage bucket reachable with the public key or a user session. Enumerate from code and config, not from docs, and cross-check against docs.

## Columns
Endpoint (method and path, or object name) | Purpose | Auth mechanism (bucket) | Authorisation applied (none, signed-in only, role, ownership) | Data touched | Data risk exposure | Side effects (writes, deletes, external calls, AI spend, sends mail) | Notes and weak spots

## Auth mechanism buckets (group the table by these, weakest first)
1. **None (public).** No credential needed. Includes pages that fetch data without a session and anon-key access.
2. **Shared secret or API key.** One static value in a header or query (cron, webhook, admin, partner or integration key). Note: compared timing-safely, fails closed if unset, scoped or all-powerful, who holds it.
3. **User session or bearer token, signed-in only.** Verified token but no role or ownership check beyond "is a member".
4. **User session with role or ownership checks.** Per-role, per-permission, or row-level security that depends on the user.
5. **Service-to-service or signed request.** Webhook signature, signed link, mutual secrets with a timestamp.
6. **Browser-direct data access.** Table, function or bucket reached from the browser with the public key and the user's session. Governed entirely by row-level security and grants; list the policy type (open, flat rule for every signed-in user, role-based).
7. **Internal only.** Cron or job entry points not exposed, or reachable only with a cron secret.

If one endpoint accepts more than one mechanism (cron secret or any signed-in user), file it under the weaker mechanism and say so.

## Data risk exposure (rate each endpoint by what a caller can reach, not by what the screen shows)
- **Critical:** credentials or tokens; payment, bank or other financial account details; special-category or sensitive personal data at scale; the ability to export or dump a whole dataset; destructive action over many records.
- **High:** personal or commercial data about many individuals or customers; bulk write or delete; sends mail or messages; spends money (AI calls, paid APIs); acts with a privileged token.
- **Medium:** internal business data per record; single-record writes; operational settings.
- **Low:** non-sensitive reference data or configuration.
- **None:** static content.
Within each bucket, sort the rows by exposure, highest first.

## Reading the table (what to flag)
- Any Critical or High exposure in bucket 1, 2 or 3.
- Any bucket 2 secret that covers broad data or is shared across several apps.
- Bucket 3 endpoints that touch Critical data with no role check.
- Identity taken from the request (user IDs, owner or actor fields) instead of the verified session, on bucket 2 or 3 endpoints.
- Bucket 6 objects with open or flat policies on Critical or High data.
- Endpoints that send a token or request to a caller-supplied address.
- Endpoints missing from docs, or documented gates that the code does not apply.

## Output
1. A summary count by bucket and by exposure (a small matrix).
2. The full table, grouped by bucket then ranked by exposure.
3. A short list of the weak spots found, each tied to a finding.
Do not include secret values. Name a secret by variable name only.
