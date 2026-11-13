# Platform settings (the repo files cannot show these)

Read the GitHub rows yourself with the `gh api` call given, when the user agreed at intake: GET only, never a write, with their existing `gh` login. Substitute `{repo}` with `owner/name`. A 403 or 404 that means "no access", rather than "not set", makes that row To verify. Confirm endpoint names against current GitHub documentation if a call fails unexpectedly.

Rows with no `gh api` call (the host, the registry, cloud IAM), and GitHub rows you could not read, go in the report as: **To verify:** do this (where). **Expect:** result. **If not:** meaning and next step.

| Setting | Read with (GitHub) or where to check | Expect |
|---|---|---|
| Branch protection on main | `gh api repos/{repo}/branches/main/protection` (a 404 "Branch not protected" means none), and `gh api repos/{repo}/rulesets` then `rulesets/{id}` for each. A ruleset with `enforcement: disabled` protects nothing | If PRs are used: PR required, required status checks (the gate jobs), no force push, no deletion. If direct pushes are the practice: say so and confirm no force push |
| Required checks match the workflow job names | Same calls: `required_status_checks` | Every gate is required, and the names match the jobs exactly. Path-filtered workflows need an always-running summary job, or the required check blocks unrelated PRs |
| Environments | `gh api repos/{repo}/environments`, and for each one `environments/{name}/deployment-branch-policies` | Production has required reviewers or a wait timer (on itself or on an approval environment its jobs need). Production's branch policy allows only main: a `refs/pull/*` pattern lets PR code read production secrets. The pre-production environment has its own secrets and its own database. Two environments is enough; flag a third as too many to maintain |
| Secret scope | `gh api repos/{repo}/actions/secrets` and `environments/{name}/secrets` (names only) | Production credentials live in the production environment only, not repo-wide |
| Default workflow token permissions | `gh api repos/{repo}/actions/permissions/workflow` | `default_workflow_permissions: read`. If it is read, a workflow with no `permissions:` block is not a gap |
| Allowed actions | `gh api repos/{repo}/actions/permissions` (and `/selected-actions` if selected) | Restricted to GitHub, verified creators or a named list where possible |
| Fork PR workflows | GitHub > Settings > Actions > General | Workflows from outside contributors need approval; secrets are not passed to forks |
| Dependabot alerts | `gh api repos/{repo}/vulnerability-alerts` (204 means on); open alerts with `gh api "repos/{repo}/dependabot/alerts?state=open&per_page=100"` grouped by `security_advisory.severity` and `created_at` | On, and no High or Critical open longer than 30 days |
| Dependabot security updates | `gh api repos/{repo}/automated-security-fixes` | `enabled: true`, so fix PRs open automatically. `dependabot.yml` cannot show this: it configures version updates only |
| Dependabot version updates | `.github/dependabot.yml` | Every ecosystem in the repo, including Docker and GitHub Actions |
| Secret scanning and push protection | `gh api repos/{repo}` → `security_and_analysis` | On where available. For private repos it is a paid add-on: if off, mark it n/a and rely on gitleaks (C26, H3) |
| Branch auto-delete | `gh api repos/{repo}` → `delete_branch_on_merge` | `true`. If on, old branches are a one-off cleanup, not a practice gap |
| Hosting auto-deploy | The host's project settings (for example Vercel > Git, Render > Settings) | Automatic production deploy off if the pipeline deploys; one controlled path to production |
| Hosting environment variables | The host's project settings | Values scoped per environment; no production values in preview or development |
| Hosting pre-deploy and health checks | The host's service settings | Any migration or pre-deploy command the pipeline relies on is set; health check path set |
| Deploy hooks and tokens | The host and GitHub | Tokens in environment secrets; hooks not exposed in the repo |
| Registry settings | The registry (for example GitHub Packages > package settings) | Deploys use digests or immutable tags; retention set; only this repo can write |
| Cloud and model-provider trust | Cloud IAM, or the provider's console for OIDC federation | Trust limited to the repo and, ideally, the one workflow that needs it; no long-lived keys |
| Notifications | GitHub or a chat integration | A named person or channel is told when a deploy fails |
