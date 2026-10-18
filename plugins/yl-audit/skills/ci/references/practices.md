# Practice checks

Rate each: **pass**, **partial**, **fail**, **n/a for stage**, or **unverified** (needs a setting or a platform check). Cite `file:line`.

Stages S1, S2 and S3 are defined in `../../full/references/rules.md`.

## Checks

| # | Check | What to look for | S1 | S2 | S3 | Typical rating if absent |
|---|---|---|---|---|---|---|
| C1 | Lint | `eslint`, `ruff`, `golangci-lint` or similar in a workflow step | Expected | Expected | Expected | Low to Medium |
| C2 | Format check | `prettier --check`, `black --check`, `gofmt -l` | Expected | Expected | Expected | Low |
| C3 | Typecheck | `tsc --noEmit`, `mypy`, `pyright` | Expected | Expected | Expected | Medium |
| C4 | Tests | test command run in CI, failing the job | Expected | Expected | Expected | Medium |
| C5 | Dependency audit | `npm audit`, `pip-audit`, `osv-scanner`, Dependabot alerts gating | Expected | Expected | Expected | Medium |
| C6 | Build | the app builds in CI (not only on the host) | Expected | Expected | Expected | Medium |
| C7 | Gates on push to main | workflow `on: push: branches: [main]` runs C1 to C6 | Expected | Expected | Expected | Medium |
| C8 | Pull requests | PR workflow or branch rules exist | Optional (if coverage is trusted) | Expected | Expected | Low (S1), Medium (S2+) |
| C9 | Gates on PRs, required before merge | `on: pull_request` runs the gates; required checks configured (verify in settings) | n/a if no PRs | Expected | Expected | Medium |
| C10 | Automated PR review | a workflow, app or config that reviews every PR automatically | n/a if no PRs | Expected | Expected | Low |
| C11 | Short-lived branches | few branches, recent, merged quickly; no long-lived feature branches | Expected | Expected | Expected | Low |
| C12 | Build once, promote artifact | image or package built and pushed in CI, tagged by commit, same artifact deployed to each environment | Expected (pragmatic if static site) | Expected | Expected | Medium (host builds from git) |
| C13 | Registry hygiene | immutable tags or digests, retention, scanning | Optional | Expected | Expected | Low |
| C14 | Pre-production environment (staging or test, same thing) | one exists with its own database and secrets, and deploys automatically on push or merge to main; at most one such environment beyond production (a third is too many for most teams: do not recommend it) | Expected, or planned soon if absent | Expected | Expected | Low (S1, S2: say it is needed soon), Medium (S3) |
| C15 | Production trigger | manual dispatch or environment approval, automated in the pipeline | Expected | Expected | Expected | Medium |
| C16 | Same artifact to prod | production deploys the artifact proven in the pre-production environment, not a rebuild | Expected | Expected | Expected | Medium |
| C17 | Rollback | previous version kept; documented or scripted rollback | Expected | Expected | Expected | Low to Medium |
| C18 | Pre-deploy migrations | migration step runs automatically before the app deploy, dedicated credential, applied-state tracked | Expected | Expected | Expected | Medium |
| C19 | Migration safety | expand then contract; backward compatible with the code still running; tested on a disposable DB in CI; drift check | Expected | Expected | Expected | Low to Medium |
| C20 | Feature flags | a flag library or config, a kill switch, deploy decoupled from release | Optional | Optional | Expected where releases are risky | none (a win when present) |
| C21 | Environment variables | config read from env; per-environment values; example file; required values validated at startup; no hard-coded config | Expected | Expected | Expected | Medium (hard-coded secrets: High) |
| C22 | Secrets handling | environment-scoped secrets, not repo-wide where avoidable; none in workflow inputs, logs or the repo | Expected | Expected | Expected | High if exposed |
| C23 | Pipeline supply chain | Actions pinned to commit hashes; `permissions:` read-only by default; no `pull_request_target` that checks out PR code with secrets; OIDC instead of long-lived cloud keys; short artifact retention; images by digest | Expected | Expected | Expected | High (untrusted code can reach secrets), Low (tags) |
| C24 | Reliability of the pipeline | `concurrency` groups, job timeouts, failure notifications | Optional | Expected | Expected | Low |
| C25 | Dependency automation | Dependabot or Renovate covering every ecosystem | Expected | Expected | Expected | Low |

## Severity cues
- **High:** untrusted code can get secrets or deploy (for example `pull_request_target` with the PR head checked out, or secrets passed to fork PRs); a long-lived production credential exposed in the repo or logs.
- **Medium:** no gates on main for a production app; production deploys with no approval; migrations applied by hand; host-built deploys where each environment builds differently; production secrets scoped repo-wide to every workflow; required checks missing when PRs are used.
- **Low:** no pre-production environment for a small team (rate as needed soon, not as a gap to fill with more environments); no feature flags; Actions on tags; long artifact retention; no Dependabot; branch hygiene; no automated PR review.
- Anything rated down because of stage gets the pragmatic-suggestion label from the rules.

## Evidence patterns
Look for these strings in workflows and scripts:
- Gates: `lint`, `prettier`, `tsc`, `typecheck`, `test`, `vitest`, `jest`, `pytest`, `npm audit`, `pip-audit`, `osv-scanner`, `build`.
- Artifacts: `docker build`, `docker push`, `ghcr.io`, `docker/build-push-action`, `registry`, image tags by `${{ github.sha }}`.
- Deploy: `workflow_dispatch`, `environment:`, `needs:`, `vercel deploy`, `fly deploy`, `kubectl`, `helm`.
- Migrations: `migrate`, `drizzle-kit`, `prisma migrate deploy`, `supabase db push`, `alembic upgrade`, `flyway`.
- Hygiene: `uses:` lines (tag versus 40-character hash), `permissions:`, `pull_request_target`, `secrets.`, `id-token: write`, `retention-days`, `concurrency`, `timeout-minutes`.
- PR review: `claude-code-action`, `coderabbit`, `copilot`, `danger`, `.github/pull_request_template.md`, `CODEOWNERS`.
- Flags: `launchdarkly`, `unleash`, `growthbook`, `posthog` feature flags, a `flags` or `features` config module.
- Hosting auto-build signals: `vercel.json`, `netlify.toml`, `render.yaml` with build commands and no CI deploy job.
