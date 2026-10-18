# Settings to verify (the repo cannot show these)

Give the user these checks. Do not run them. Confirm names and paths against current GitHub and hosting documentation. Each unverified item in the report uses: **To verify:** do this (where). **Expect:** result. **If not:** meaning and next step.

| Setting | Where | To verify: do this | Expect |
|---|---|---|---|
| Branch protection or rulesets on main | GitHub > Settings > Rules / Branches | Open the ruleset for main | If PRs are used: required status checks (the gate jobs), no force push, no deletion. If direct pushes are the practice: say so and confirm no force push |
| Required checks match the workflow job names | same | Compare required check names with the jobs in `.github/workflows` | Every gate is required; names match exactly |
| Environments | GitHub > Settings > Environments | Open production and the pre-production (staging or test) environment | Production has required reviewers or a wait timer and its own secrets; the pre-production environment has its own secrets and its own database. Two environments is enough; flag a third as too many to maintain |
| Secret scope | GitHub > Settings > Secrets and variables | List repository versus environment secrets (names only) | Production credentials live in the production environment only, not repo-wide |
| Default workflow token permissions | GitHub > Settings > Actions > General | Read the workflow permissions setting | Read-only by default |
| Allowed actions | GitHub > Settings > Actions > General | Read the policy | Restricted to approved or pinned actions where possible |
| Fork PR workflows | same | Read fork pull request approval settings | Workflows from outside contributors require approval; secrets are not passed to forks |
| Hosting auto-deploy | the host's project settings (for example Vercel > Git) | Read whether production deploys on every push and which branch is production | Automatic production deploy off if the pipeline deploys; one controlled path to production |
| Hosting environment variables | the host's project settings | List variable names by environment (not values) | Values scoped per environment; no production values in preview or development |
| Deploy hooks and tokens | the host and GitHub | List deploy hooks and who holds deploy tokens | Tokens in environment secrets; hooks not exposed in the repo |
| Registry settings | the registry (for example GitHub Packages) | Read tag immutability, retention, scanning | Immutable tags or digests used for deploys; retention set; scanning on |
| Cloud trust | cloud IAM | Read the GitHub trust relationship | OIDC trust limited to the repo and environment; no long-lived keys |
| Dependabot | GitHub > Settings > Code security | Read alerts and update settings | Alerts and version updates on for every ecosystem |
| Notifications | GitHub or chat integration | Confirm who is told when a workflow fails | A named person or channel |
