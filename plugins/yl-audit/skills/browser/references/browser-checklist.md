# Browser checklist

## Headers to record (page and a static file)
- Content-Security-Policy (script-src, style-src, connect-src, frame-ancestors; flag `unsafe-inline`, `unsafe-eval`, wildcards)
- X-Frame-Options or frame-ancestors
- Strict-Transport-Security
- X-Content-Type-Options
- Referrer-Policy, Permissions-Policy
- Access-Control-Allow-Origin (and Allow-Credentials); `*` on data is a finding, `*` on static files usually is not
- Cache-Control on authenticated pages (should not be cached publicly)
- Set-Cookie flags: Secure, HttpOnly, SameSite

## Signed-out probes (GET, static or known-safe paths only)
- `/`, a deep link, `/login`, `/admin` style paths the code or navigation shows
- `/.env`, `/.git/config`, `/package.json`, `/*.map` (expect not found)
- A nonsense path (expect the app shell or a plain 404, no stack trace)
- `robots.txt`, `sitemap.xml`
- If an access gateway or single sign-on fronts the site: signed out, request a data or asset file directly, and the host's default or preview address (for example `*.vercel.app`, `*.netlify.app`, `*.pages.dev`) if it is known. Expect the gateway every time, not the content.
- The site's own API paths only if the code review shows them to be pure reads

## Bundle inspection (counts only)
- Count matches, never print: Supabase project addresses, JWT-shaped strings, `sb_publishable_`, `sb_secret_`, `service_role`, `AIza`, `sk-`, private key headers.
- Public environment variable names visible in the bundle.

## Network log fields
Method | Host | Path | Status | Credentials sent (none, bearer, cookie, API key header) | Response shape (field names only) | Destination class (own API, database direct, storage, identity provider, third party) | Notes

## Storage record
Where (local, session, IndexedDB, cookie) | Key name | Size | Kind (token, cache, preference) | Token kind and claim names | Survives sign-out?

## Reading a token safely
Report only: which token it is (identity, access, refresh), issuer host, audience, expiry window, scope names, role claim. Never the signature or the full string. Use a read-only snippet that decodes the payload and prints claim keys and these few values only.

## Read-only snippets
Only snippets that read: list storage key names and lengths, list cookie names, count matches. No snippet may write, click, submit, call fetch to a mutating route, or print a token.

## Sign-out test (with consent)
After signing out: are the local and session storage entries gone, do cached screens or the back button show data, does the next sign-in show the previous user's data.

## Stop and report
Stop and tell the user the path and action when a step would save, submit, delete, send, create, approve, change a setting, or sign in or out without their say.
